# 日志冷热架构：大文件 + ES 查询 + Nginx 限流

> **主查询路径**：前端按 **userId + 时间范围** 捞日志，并可叠加条件（如 `category=network` 网络日志、level、path 等），时间长则**分页**。  
> **详情路径**：列表点进某条后，用 **logId** 取全文（ES 有则直接用，没有再回源大文件并回填）。  
> 后台 API 以 **Node.js** 为例。本文含：整体查询链路、条件筛选、分页、点查回填、Nginx、Node 难点。

---

## 1. 架构定位

| 层 | 角色 |
|----|------|
| 大文件系统 | **原文真相源**，只写追加；按 **logId** 随机读全文 |
| Elasticsearch | **检索主入口**：按 userId、时间、类别等过滤 + 分页；存摘要/部分字段，可 ILM 过期 |
| API 服务 | **列表走 ES search**；**详情走 ES get，miss 再回源文件回填** |
| Nginx | TLS、路由、多用户/查询接口限流 |

```text
前端控制台
    → Nginx
    → Log Query API（Node.js）
         │
         │ ① 主路径：条件列表查询
         ├─ GET /api/logs?userId&from&to&category=network&level&...&cursor=
         │     → 鉴权 → 拼 ES bool filter → search_after 分页
         │     → 返回摘要列表（logId, ts, category, preview...）
         │
         │ ② 详情路径：点开某一条
         └─ GET /api/logs/{logId}
               → ES.get(logId)
                    hit  → 返回（或摘要+fileUri 再拉全文）
                    miss → 大文件系统.get(logId) → 回填 ES → 返回
```

### 1.1 主从关系（务必记清）

| 能力 | 靠谁 | 说明 |
|------|------|------|
| 按 userId + 时间 + 条件筛选 | **ES** | 冷文件无法高效「按用户扫」 |
| 分页翻页 | **ES search_after** | 见第 5 节 |
| 单条全文 | ES 或 **大文件** | 列表先出摘要；详情可回源 |
| 写入管线 | 上报/MQ → 文件存原文 + **异步索引进 ES** | 仅点查回填撑不住列表 |

原则：

- 列表/筛选 **不以 logId 为入口**，以 **userId + 时间（必选）+ 业务条件（可选）** 为入口。  
- logId 是结果里的主键，用于详情与回源。  
- ES 挂掉时：列表降级（失败提示）；详情仍可尝试直读文件（若调用方已知 logId）。

---

## 2. 整体日志查询链路（端到端）

### 2.1 写入（查询的前提）

```text
客户端/业务上报
  → 接入服务（鉴权、限流、batchId 幂等）
  → 原文写入大文件系统（key=logId）
  → 异步：抽取索引字段写入 ES
       userId, ts, logId, category, level, path, statusCode, msgPreview, fileUri...
```

没有「索引字段进 ES」，后面所有筛选/分页都做不了。

### 2.2 查询（控制台用法）

```text
1. 用户选时间范围（必选）+ 默认带当前 userId（或管理员选用户）
2. 可选筛：日志类型/category（network）、level、URL path、statusCode、关键字…
3. 点「查询」→ 列表 API 第一页
4. 滚动/下一页 → 带同一筛选条件 + cursor
5. 点某一行 → 详情 API（logId）拉全文
```

### 2.3 详情读路径（logId，辅路径）

```text
GET /api/logs/{logId}

1. 鉴权（该 log 是否属于可访问用户）
2. ES 点查
3. hit → 返回；若只有摘要则按 fileUri 读文件拼全文
4. miss → singleflight/锁 → 文件服务 → 回填 ES → 返回
```

可选：异步回填；极热 logId 短缓存。并发见第 3 节。

---

## 3. 并发：多请求同时查同一个 logId

### 3.1 问题

同一 `logId` 在 ES 未命中时，多个用户/标签页同时请求 → 集体 miss → 重复打文件服务 → 并发写 ES。

### 3.2 后端解决方案（主方案）

| 手段 | 做法 |
|------|------|
| **Singleflight / 互斥** | 同一 `logId` 只允许一个回源；其它等待或复用结果 |
| **分布式锁** | 多实例时用 `SET NX EX backfill:{logId}`，拿到锁的才回源写 ES |
| **幂等写 ES** | `_id = logId`；`index` 覆盖或 `create` 遇 409 忽略 |
| **routing** | 写入与点查带同一 `routing=logId`，降低扫分片 |

伪流程：

```text
miss 后：
  if tryLock("backfill:"+logId):
      body = file.get(logId)
      es.index(_id=logId, body)
      unlock
  else:
      sleep 短暂 / 轮询 es.get
      或自己读文件返回但不写 ES（降级）
```

### 3.3 Nginx 能不能「合并同 logId」？

**标准 Nginx 不能把多个相同 URI 的请求合成一次上游调用。**  
同 logId 的惊群，**必须在应用层**做 singleflight/锁；Nginx 负责的是**别让总流量把入口打垮**（见第 7 节）。

若要用缓存挡重复读：可用 `proxy_cache` 按 URI 缓存**成功响应**（见 7.3），在短 TTL 内减轻 API/ES，但 **miss 回源并发**仍靠后端互斥最稳。

---

## 4. ES 查询慢（列表 + 点查）

| 场景 | 做法 |
|------|------|
| 点查 logId | `GET _doc/{logId}`，勿用 match 扫全文 |
| 列表 userId+时间+条件 | **filter** 组合；按天缩索引；见第 5 节分页 |
| `_source` 太大 | 列表只要摘要；全文回源文件 |
| 列表 routing | 主路径按 userId 查时，写入可用 `routing=userId` |
| 深分页 | **禁止**大 from；用 search_after |
| 条件过多/乱 match | 枚举字段用 term；关键字单独 multi_match 且限制字段 |

---

## 5. 主查询：userId + 时间 + 条件筛选 + 分页

### 5.1 场景：整个日志查询怎么走

控制台典型交互：

```text
必选：userId（或登录态默认）+ 时间范围 from/to
可选：category=network（网络日志）、level=ERROR、
      path 前缀、statusCode、keyword…
操作：查询 → 翻页 → 点某行看详情（logId）
```

API 形态：

```text
GET /api/logs
  ?userId=u1
  &from=2026-10-01T00:00:00Z
  &to=2026-10-19T00:00:00Z
  &category=network          # 条件：只要网络日志
  &level=ERROR               # 可选
  &pathPrefix=/api/pay       # 可选
  &statusCode=500            # 可选
  &q=timeout                 # 可选关键字
  &limit=50
  &cursor=...
```

痛点：时间长、再叠加筛选，命中仍可能很多 → 必须分页；筛选变更要重置 cursor。

### 5.2 文档模型（筛选靠字段，不是靠扫文件）

写入 ES 时就要抽好维度（网络日志示例）：

```json
{
  "logId": "…",
  "userId": "u1",
  "ts": "2026-10-18T12:01:02.123Z",
  "category": "network",
  "level": "ERROR",
  "path": "/api/pay/confirm",
  "method": "POST",
  "statusCode": 500,
  "latencyMs": 3200,
  "msgPreview": "upstream timeout…",
  "fileUri": "fs://logs/…"
}
```

| 筛选条件 | ES 用法 | 映射建议 |
|----------|---------|----------|
| userId | `term` | keyword |
| 时间 | `range` on ts | date |
| 网络日志 | `term` category=`network` | keyword |
| level | `term` / `terms` | keyword |
| path | `prefix` 或 `term` | keyword（勿乱 text） |
| statusCode | `term` | integer |
| 关键字 | `match` 仅 preview/msg 字段 | text + keyword 分离 |

`category` 枚举要在上报约定好：`network` / `crash` / `biz` / `perf`…，前端筛选项与之一致。

### 5.3 查询体：bool filter 叠条件

所有「精确过滤」进 **filter**（可缓存、不算分）；关键字才进 must/match：

```json
{
  "size": 50,
  "query": {
    "bool": {
      "filter": [
        { "term": { "userId": "u1" } },
        { "range": { "ts": { "gte": "…", "lte": "…" } } },
        { "term": { "category": "network" } },
        { "term": { "level": "ERROR" } },
        { "prefix": { "path": "/api/pay" } },
        { "term": { "statusCode": 500 } }
      ],
      "must": [
        { "match": { "msgPreview": "timeout" } }
      ]
    }
  },
  "sort": [
    { "ts": "desc" },
    { "logId": "asc" }
  ],
  "_source": ["logId", "ts", "category", "level", "path", "statusCode", "msgPreview"]
}
```

无关键字时不要带 `must`，纯 filter 更快。

### 5.4 为什么不要用 from/size 深分页

```text
from=10000&size=50 → 越深越贵，常触 from+size 上限
```

只适合前几页预览。正式翻页用 **search_after**。

### 5.5 分页主方案：search_after 游标

排序：`ts desc` + `logId asc`（稳定、防同毫秒重复/漏）。

**下一页**带上页最后一条的 sort：

```json
"search_after": [ 1729000000123, "log_abc" ]
```

对外：

```text
响应：{ "items": [...], "nextCursor": "…" | null }
下一页：相同筛选参数 + cursor=nextCursor
```

**硬规则：**

1. cursor 不透明（前端不解析）  
2. **任一筛选条件变化（含 category、时间、level）→ 必须清空 cursor 从第一页来**  
3. cursor 内建议校验「条件哈希」，防篡改 cursor 配错筛选  

cursor 建议编码：

```text
{ filtersHash, window, searchAfter }
```

### 5.6 时间太长：时间切片 + 游标

跨很多天时，即使有筛选，仍建议：

```text
内部按天（或小时）切窗口
cursor 带上当前 window + searchAfter
本窗翻完 → 下一窗，searchAfter 清空
```

或产品限制：单次最长 1～7 天。  
**有 category 等强过滤时**，命中变少，可适当放宽跨度，但仍按天缩索引集合。

### 5.7 只查「网络日志」时的优化

- 写入侧保证 `category=network` 打标准确（客户端网络库埋点 / 网关访问日志）  
- 可选：**独立索引** `logs-network-YYYY.MM.DD`，网络控制台只搜该前缀，更省  
- 或同一索引 + term filter（实现简单，网络占比高时仍够用）  
- 列表勿返回 response body 全文；详情再按 logId 回源  

### 5.8 PIT / scroll

| API | 用途 |
|-----|------|
| search_after | 控制台翻页（主推） |
| PIT + search_after | 长翻页要快照一致、导出 |
| scroll | 后台一次性导出，不给前端 |

### 5.9 索引与性能

| 项 | 建议 |
|----|------|
| 索引范围 | 只用 from～to 覆盖的日索引，禁止无脑 `logs-*` |
| routing | 主路径按用户查：`routing=userId` 写入与查询一致 |
| limit | 封顶 50/100 |
| total | 默认不精确深 total；返回 hasMore / nextCursor |
| 限流 | 列表接口比详情更严 |

### 5.10 与冷存关系

| 步骤 | 存储 |
|------|------|
| 筛选 + 分页列表 | **仅 ES** |
| 点开详情全文 | ES 摘要不足时 → **大文件** by logId |

上报必须异步写 ES 索引字段；否则「网络日志」筛不出来。

### 5.11 Node.js 列表示意（含条件）

```js
function buildFilters({ userId, from, to, category, level, pathPrefix, statusCode }) {
  const filter = [
    { term: { userId } },
    { range: { ts: { gte: from, lte: to } } },
  ];
  if (category) filter.push({ term: { category } });       // network
  if (level) filter.push({ term: { level } });
  if (pathPrefix) filter.push({ prefix: { path: pathPrefix } });
  if (statusCode != null) filter.push({ term: { statusCode } });
  return filter;
}

async function listLogs(params) {
  const pageSize = Math.min(params.limit ?? 50, 100);
  const filtersHash = hashFilters(params); // 变条件则旧 cursor 作废
  const decoded = decodeCursor(params.cursor);
  if (decoded && decoded.filtersHash !== filtersHash) {
    throw badRequest("cursor does not match filters");
  }
  const window = decoded?.window ?? firstWindow(params.from, params.to);
  const searchAfter = decoded?.searchAfter;

  const body = {
    size: pageSize,
    query: {
      bool: {
        filter: buildFilters({ ...params, from: window.start, to: window.end }),
        ...(params.q ? { must: [{ match: { msgPreview: params.q } }] } : {}),
      },
    },
    sort: [{ ts: "desc" }, { logId: "asc" }],
    _source: ["logId", "ts", "category", "level", "path", "statusCode", "msgPreview"],
  };
  if (searchAfter) body.search_after = searchAfter;

  const resp = await es.search({
    index: indicesForWindow(window),
    routing: params.userId,
    body,
    request_timeout: "3s",
  });

  const hits = resp.hits.hits;
  let nextCursor = null;
  if (hits.length === pageSize) {
    nextCursor = encodeCursor({
      filtersHash,
      window,
      searchAfter: hits[hits.length - 1].sort,
    });
  } else {
    const w = nextWindow(window, params.from, params.to);
    if (w) nextCursor = encodeCursor({ filtersHash, window: w, searchAfter: undefined });
  }
  return { items: hits.map(unwrap), nextCursor };
}
```

### 5.12 方案对照

| 方案 | 适用 |
|------|------|
| filter + search_after | 条件筛选 + 翻页（主推） |
| 时间切片 + 游标 | from/to 很长 |
| 限制最大时间跨度 | 产品可接受时最省心 |
| 网络日志独立索引 | network 流量大、控制台专用 |
| PIT | 导出/长快照 |
| from/size | 仅前几页 |

### 5.13 本节收口

> **整个查询**：写入时原文进文件、维度进 ES → 列表用 **userId+时间+条件（如 network）** 做 filter 查询并 **search_after 分页** → 点详情用 **logId** 取全文（可回源）。时间长则切窗；改筛选必须重置 cursor。

---

## 6. ES 过期时间怎么设

ES 日志热层**优先用「按时间索引 + ILM」**，而不是给每个 doc 设 Redis 式 TTL。

### 6.1 推荐：滚动索引 + ILM

```text
logs-2026.10.18
logs-2026.10.19   ← 回填写入当前热索引（别名 logs-hot）
...
超过 N 天 → ILM delete 整索引
```

再点查 miss → 再回源回填，形成闭环。  
列表可查范围也应与 ILM 保留天数对齐（或产品提示「仅支持近 N 天」）。

### 6.2 N 怎么选

| 因素 | 倾向 |
|------|------|
| 命中率高、回源文件贵/慢 | N 大（14～30 天） |
| ES 磁盘紧、文件读便宜 | N 小（3～7 天） |
| 业务「热查只要一周」 | N=7，对齐产品 |

结合指标调：`es_hit_rate`、回源 QPS、磁盘水位。

### 6.3 备选

- 文档带 `indexedAt`，定时 `delete_by_query`（比删索引重）  
- 逻辑过期：读时看 `cachedAt`，过期可当 miss 或异步刷新  

---

## 7. Nginx：多用户查询 & 同 logId 怎么设置

职责边界：

| 诉求 | Nginx | 应用 |
|------|-------|------|
| 很多用户同时查（总流量） | **限流、连接数、超时** | 线程池/连接池 |
| 同一 logId 并发 miss 回填 | 可辅助短缓存 | **互斥 / 幂等（主）** |
| 列表查询（userId+时间） | 单独更严限流 | search_after + 时间切片 |
| 鉴权 | 可粗拦 | JWT/权限（主） |
| 路由 | `location` 转发 | 业务路由 |

### 7.1 多用户查询：限制总入口流量

防止全体用户把 API/ES/文件服务打满：

```nginx
# 限制每个 IP 的请求速率（多用户场景的基础保护）
limit_req_zone $binary_remote_addr zone=log_by_ip:10m rate=10r/s;

# 限制并发连接（可选）
limit_conn_zone $binary_remote_addr zone=conn_by_ip:10m;

upstream log_api {
    server 127.0.0.1:8080;
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name log.example.com;

    location /api/logs/ {
        limit_req zone=log_by_ip burst=20 nodelay;
        limit_conn conn_by_ip 20;

        proxy_pass http://log_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;

        proxy_connect_timeout 2s;
        proxy_send_timeout 5s;
        proxy_read_timeout 5s;   # 与后端超时预算对齐
    }
}
```

说明：

- `rate=10r/s` + `burst`：按 **IP** 平滑多用户流量；内部办公网可改大或按账号限流（见下）  
- 超时要**短于/对齐**后端对文件服务的预算，避免 Nginx 已断、后端还在回源  

### 7.2 按登录用户限流（多用户更公平）

只按 IP 时，NAT 下多人会误伤。可把用户维度传给 Nginx：

```nginx
# 假设鉴权后由 auth 子请求或网关注入；示意用 header
map $http_x_user_id $limit_key {
    default $http_x_user_id;
    ""      $binary_remote_addr;   # 未登录回落 IP
}

limit_req_zone $limit_key zone=log_by_user:10m rate=5r/s;
```

更常见初创做法：**Nginx 按 IP 粗限流 + 应用按 userId 细限流**。

### 7.3 同一 logId：Nginx 侧能做的

**1）按 URI 限流（防单个 id 被刷）**

```nginx
# 对「具体日志路径」单独限速，避免同一 logId 被打爆
limit_req_zone $request_uri zone=log_by_uri:10m rate=20r/s;

location ~ ^/api/logs/[^/]+$ {
    limit_req zone=log_by_ip burst=20 nodelay;
    limit_req zone=log_by_uri burst=40 nodelay;  # 叠加：同 id 全网共享额度
    proxy_pass http://log_api;
    # ... 同上 proxy 头与超时
}
```

注意：`$request_uri` 含 query string 时要统一；纯 `/api/logs/{id}` 最合适。  
这是**限流**，不是合并回源；后端仍要 singleflight。

**2）短时 proxy_cache（可选，挡重复命中）**

仅缓存**成功的 GET**，减轻重复打 API（尤其已回填后）：

```nginx
proxy_cache_path /var/cache/nginx/logs levels=1:2 keys_zone=log_cache:20m
                 max_size=2g inactive=10m use_temp_path=off;

location ~ ^/api/logs/[^/]+$ {
    proxy_cache log_cache;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    proxy_cache_valid 200 30s;      # 短 TTL；权限敏感慎用
    proxy_cache_methods GET HEAD;
    add_header X-Cache-Status $upstream_cache_status;

    # 有用户态权限时：要么不缓存，要么 cache key 带上 user_id
    # proxy_cache_key "$http_x_user_id:$request_uri";

    limit_req zone=log_by_ip burst=20 nodelay;
    proxy_pass http://log_api;
}
```

权限注意：

- 日志若**按用户隔离**，`proxy_cache_key` **必须带 userId/角色**，否则会串数据  
- 不确定时：**关闭 proxy_cache**，只靠应用限流 + ES  

**3）不要指望 Nginx 解决「并发回填」**  
锁与 singleflight 放在 API；Nginx 只降总 QPS、降同 URI 毛刺。

### 7.4 多用户 + 同 logId 的分工小结

```text
多用户同时查不同日志
  → Nginx：per-IP / per-user 限流 + 超时 + keepalive
  → API：正常点查 ES；miss 各自回源（可全局限流回源）

多用户同时查同一个 logId
  → Nginx：per-URI 限流 +（可选）短 proxy_cache
  → API：singleflight/分布式锁 + ES _id 幂等写  ← 解决惊群回源的关键
```

---

## 8. 端到端时序（同 logId 三用户同时 miss）

```text
UserA/B/C → Nginx（URI 限流通过）
         → API 均 ES miss
         → A 拿到 backfill 锁 → 读文件 → 写 ES → 返回
         → B/C 等锁或轮询 ES → hit → 返回
         （若开了 proxy_cache，后续短时间直接在 Nginx 命中）
```

---

## 9. Node.js 后台难点（本场景专项）

Node.js 适合 IO 密集的「查 ES / 读文件 / 写 ES」，但单线程事件循环 + 默认内存模型会在这套冷热架构里踩坑。下面按**会出事的点**写。

### 9.1 事件循环被堵住（CPU / 同步重活）

| 风险 | 表现 | 对策 |
|------|------|------|
| 大日志 JSON.parse / 正则脱敏在请求里同步做 | 延迟尖刺、全进程 QPS 掉 | 限制正文大小；大处理丢 Worker Thread / 独立清洗服务；只把摘要进 ES |
| 压缩/加解密/超大 stringify | 同上 | 流式处理；能不下载全文就不下载 |
| 同步 fs / 同步 crypto | 卡死 loop | 一律 async API |

面试句：Node 擅长等网络，不擅长在请求路径里啃 CPU。

### 9.2 并发回填：进程内 singleflight + 多实例锁

Node 单进程内可用内存 Map 做 **singleflight**（同 logId 共用一个 Promise）：

```js
// 示意：同 logId 合并成一次回源
const inflight = new Map();

async function getLog(logId) {
  const cached = await esGet(logId);
  if (cached) return cached;

  if (inflight.has(logId)) return inflight.get(logId);

  const p = (async () => {
    const locked = await redis.set(`backfill:${logId}`, "1", "EX", 30, "NX");
    try {
      if (!locked) {
        await sleep(50);
        return (await esGet(logId)) ?? fileGet(logId); // 降级：读文件可不写 ES
      }
      const body = await fileGet(logId);
      try { await esIndex(logId, body); } catch (e) { metrics.esWriteFail++; }
      return body;
    } finally {
      inflight.delete(logId);
      if (locked) await redis.del(`backfill:${logId}`);
    }
  })();

  inflight.set(logId, p);
  return p;
}
```

难点：

- **多进程 / 多机**：仅靠 Map 不够，要 **Redis SET NX**（或 etcd）  
- **锁释放**：进程崩溃要靠锁 TTL；业务超过 TTL 要续期或接受偶发重复回源  
- **Promise 泄漏**：`finally` 里必须 `inflight.delete`  

### 9.3 超时、取消与「Nginx 已断开」

| 难点 | 对策 |
|------|------|
| 文件服务慢，Node 还在等 | `AbortController` + 超时（如 2s）；与 Nginx `proxy_read_timeout` 对齐 |
| 客户端取消，回填仍继续 | 区分「响应已取消」与「回填是否继续」；可改为队列异步回填 |
| 无超时的 fetch/axios | 所有出口（ES、文件、Redis）必须设 timeout |

超时预算示例：

```text
Nginx read 5s
  └─ API 总预算 4s
       ├─ ES get  ≤ 200ms
       ├─ 文件    ≤ 2s
       └─ ES index≤ 500ms（可异步，失败不影响返回）
```

### 9.4 连接池与「打开太多连接」

Node 默认对上游并发高时易打满：

- `http.Agent` / `undici` / ES client：设 `keepAlive`、`maxSockets` / 连接上限  
- ES 官方 Client、文件服务 HTTP 客户端都要**显式池化**，并计入连接池总账  
- 避免每个请求 `new Client()`  

### 9.5 大 body 与内存

| 难点 | 对策 |
|------|------|
| 一次把几百 MB 日志读进 Buffer | 流式读取；ES 只存截断摘要；全文跳转文件 URL（若权限允许） |
| 多个 miss 并发 × 大文件 = RSS 爆 | **限制进程内并发回源**（如 p-limit=10）；Nginx/应用双限流 |
| V8 老生代涨、GC 卡顿 | 监控 heap；拒绝超限 log 入 ES `_source` |

### 9.6 错误处理与 async 陷阱

- 未处理的 Promise rejection 可能打挂进程 → 统一封装、`error` 中间件  
- ES 写失败：**仍返回文件内容**，只打指标，勿把整请求变成 500  
- 文件 404 vs ES 5xx 映射清晰，避免全变 500  

### 9.7 与集群模式

| 部署 | 注意 |
|------|------|
| `cluster` 多进程 或 K8s 多 Pod | singleflight 必须外置锁 |
| 单进程 PM2 | 同上，多实例仍要 Redis 锁 |
| 无状态水平扩 | Session 不放内存；限流可 Nginx + Redis |

### 9.8 观测（Node 特别要补的）

- `event loop lag`（卡顿秒杀一切优化）  
- 活跃句柄 / 连接数、heap used  
- `es_hit`、`backfill_inflight`、`file_rt_p99`、`es_write_fail`  
- 结构化日志 + `request_id`（与 Nginx `$request_id` 透传）  

### 9.9 Node 技术选型提示（本场景）

| 能力 | 常见选择 |
|------|----------|
| HTTP 框架 | Fastify / Express + 超时中间件 |
| ES | `@elastic/elasticsearch`（关掉乱重试或限制重试） |
| 锁 / 短缓存 | ioredis |
| 并发上限 | p-limit / bottleneck |
| 指标 | prom-client |

ES 客户端注意：默认重试可能在故障时**放大流量**；对点查与回填要收敛重试策略（呼应弹力：重试 + 熔断 + 幂等）。

### 9.10 Node 难点一句话

> IO 模型合适，但要用 **singleflight+Redis 锁** 防同 id 惊群，用 **超时/连接池/回源并发上限** 防打爆自己和上游，用 **流式与摘要** 防大日志撑爆内存，并盯 **event loop lag**。

---

## 10. 难点汇总（面试用）

| 难点 | 对策 |
|------|------|
| 并发 miss 打穿文件服务 | 互斥回填 + `_id` 幂等 |
| 多用户总流量过大 | Nginx `limit_req` / `limit_conn` + 应用限流 |
| 单 logId 被刷 | Nginx 按 `$request_uri` 限流 |
| ES 点查慢 | Get by id、routing、控 `_source`、ILM 控索引体量 |
| **userId+长时间列表分页** | **search_after 游标**；**时间切片**；禁深 from/size；按天缩索引 |
| ES 过期 | 按天索引 + ILM 保留 N 天（与可查窗口对齐） |
| 缓存串权 | 带权限则勿盲目 proxy_cache，或 key 含 userId |
| ES 写失败 | 仍返回文件内容；指标告警 |
| 穿透非法 id | 校验 + 负缓存 |
| **Node 阻塞 loop** | 禁止请求路径重 CPU；大日志截断/Worker |
| **Node 多实例惊群** | Map singleflight + Redis NX 锁 |
| **Node 内存/连接打满** | 回源并发上限 + Agent 连接池 + 超时预算 |

---

## 11. 面试收口

> 大文件是只写真相；ES 是可 ILM 过期的热层。点查 Cache-Aside 走 `_id`；同 logId 惊群靠互斥回填。  
> **按 userId+时间列表**：filter 查询 + **search_after（ts+logId）** 不透明 cursor；跨度大则 **按天/按小时切窗口** 或限制最大时间范围；需要稳定快照再用 PIT。不要用 from/size 深翻页。  
> Nginx 限多用户与单 URI；Node 管 dual：锁、超时、连接池、event loop。ES 保留天数与「能搜多远」对齐。

---

## 12. 相关文档

- [02-interview-qa.md](./02-interview-qa.md) — 缓存击穿、限流、网关相关题；场景题 I4  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md) — 弹力与限流  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md) — Cache-Aside  


