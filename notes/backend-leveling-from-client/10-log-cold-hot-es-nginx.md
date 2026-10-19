# 日志冷热架构：大文件 + ES 回填 + Nginx 限流

> 场景：日志已写入**大文件系统（只写追加）**；前端按 **logId** 点查（先 ES 后回源回填）；另支持按 **userId + 时间范围** 列表检索，时间跨度大时要**分页**。  
> 后台 API 以 **Node.js** 实现为例。  
> 本文合并：点查冷热、列表分页、并发写、ES 慢查询、过期、Nginx、Node.js 难点。

---

## 1. 架构定位

| 层 | 角色 |
|----|------|
| 大文件系统 | **真相源**，只写（追加），按 logId 可读 |
| Elasticsearch | **热层 / 检索加速**，可丢可重建，带过期 |
| API 服务 | Cache-Aside：先 ES → miss 回源 → 回填 |
| Nginx | 入口 TLS、路由、**限流与连接保护** |

```text
多用户浏览器
    → Nginx（限流 / 连接 / 路由）
    → Log Query API
         ├─ GET /logs/{logId}          点查：ES → miss 回源文件 → 回填
         └─ GET /logs?userId&from&to   列表：ES search + 游标分页
```

原则：ES 挂了点查仍应能回源返回；回填失败不能吞掉文件内容。  
列表检索以 **ES 已索引文档** 为主（冷存全量扫文件不适合按用户+时间检索）；未进 ES 的冷数据靠扩大热保留或离线任务回填。

---

## 2. 读路径：接口要做的事

```text
GET /api/logs/{logId}

1. 鉴权授权（谁能看这条日志）
2. 参数校验（logId 格式）
3. ES 点查（优先 GET _doc/{logId}，可加 routing）
4. hit 且未逻辑过期 → 返回
5. miss →
     a. 互斥 / singleflight（见第 3 节）
     b. 调大文件系统读原文（超时预算）
     c. 不存在 → 404（可短负缓存防穿透）
     d. 规范化、脱敏、裁剪 _source
     e. 幂等写入 ES（_id=logId）
6. 返回正文；打点 hit/miss、回源 RT、ES 写失败
```

可选：

- 先返回文件内容，**异步**回填 ES（降首包延迟）  
- 极热 logId 再加一层 Redis 短缓存  

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
同 logId 的惊群，**必须在应用层**做 singleflight/锁；Nginx 负责的是**别让总流量把入口打垮**（见第 6 节）。

若要用缓存挡重复读：可用 `proxy_cache` 按 URI 缓存**成功响应**（见 6.4），在短 TTL 内减轻 API/ES，但 **miss 回源并发**仍靠后端互斥最稳。

---

## 4. ES 查询慢

点查 logId 应走 **Get by id**，不要用重 search。

| 原因 | 处理 |
|------|------|
| `match`/`query_string` 扫全文 | 改为 `GET /idx/_doc/{logId}` |
| `_source` 塞了整段巨大日志 | ES 只存摘要 + `fileUri`；全文回源文件 |
| 无 routing，扫全部分片 | 读写统一 `routing=logId` |
| 跨很多历史索引查 | 热数据放热索引别名；过期用 ILM 删旧索引 |
| 集群被回填打满 | 回填限速；Bulk；错峰 |
| 列表深分页 | 与点查分离；见下一节 `search_after` + 时间切片 |

---

## 5. 按 userId + 时间捞日志：分页怎么做

### 5.1 场景与约束

```text
GET /api/logs?userId=u1&from=2026-10-01T00:00:00Z&to=2026-10-19T00:00:00Z&limit=50&cursor=...
```

痛点：

- 时间跨度**很长**（几天～几周）→ 命中文档可能十万、百万级  
- 用 `from/size`（或 SQL `OFFSET`）深翻页 → ES 越往后越慢，甚至拒查  
- 跨很多按天索引时，一次 search 扫太多分片  

目标：稳定翻页、可预测延迟、保护集群。

### 5.2 为什么不要用 from/size 深分页

```text
from=10000&size=50
→ ES 每个分片先取排序靠前的大量候选再归并
→ 越深越贵，默认常限制 from+size ≤ 10000
```

只适合：**前几页预览**（如前 5 页）。管理后台「翻到很后面」必须换游标。

### 5.3 推荐方案：search_after 游标分页（主方案）

**排序键必须稳定且唯一**，常用：

```text
sort: [ { "ts": "desc" }, { "logId.keyword": "asc" } ]
```

- `ts`：业务时间  
- `logId`：同一毫秒内去重、保证游标不跳/不重  

**第一页：**

```json
{
  "size": 50,
  "query": {
    "bool": {
      "filter": [
        { "term": { "userId": "u1" } },
        { "range": { "ts": { "gte": "...", "lte": "..." } } }
      ]
    }
  },
  "sort": [
    { "ts": "desc" },
    { "logId": "asc" }
  ]
}
```

**下一页：**把上一页最后一条的 sort 值带回：

```json
{
  "size": 50,
  "query": { "...同上..." },
  "search_after": [ 1729000000123, "log_abc" ],
  "sort": [
    { "ts": "desc" },
    { "logId": "asc" }
  ]
}
```

API 对外：

```text
响应：
{
  "items": [ ... ],
  "nextCursor": "base64(ts + logId)"   // 没有下一页则 null
}

请求下一页：
GET /api/logs?userId=u1&from=...&to=...&limit=50&cursor={nextCursor}
```

服务端解码 cursor → `search_after`，**query 条件必须与第一页完全一致**（userId、时间窗、过滤项不能变）。

### 5.4 时间太长：先切时间窗，再页内游标

跨 30 天一次查，即使有 search_after，单次仍可能扫很多索引、很慢。

**时间切片（推荐与游标组合）：**

```text
用户选 from～to（例如 30 天）
API 内部按天（或按小时）切开：
  day1 → day2 → … → dayN

对外仍是一个 cursor，编码里带上：
  { windowStart, windowEnd, searchAfter }
```

翻页逻辑示意：

```text
1. 当前窗口内 search_after 还有数据 → 继续本窗口
2. 本窗口空了 → 滑到下一时间窗口（如下一天），search_after 清空
3. 所有窗口走完 → nextCursor = null
```

也可**强制产品侧限制**：单次查询跨度 ≤ 1 天 / 7 天，超了让用户改条件（最省事、最稳）。

### 5.5 Point in Time（PIT）+ search_after（页多、要稳定快照时）

长时间翻很多页时，索引持续写入可能导致「翻页过程中看到的集合在变」。需要稳定快照时：

```text
1. 开启 PIT（绑定一组索引，带 keep_alive）
2. 之后每次 search 带 pit.id + search_after
3. 结束或过期后删 PIT
```

代价：占用 ES 资源；keep_alive 要续约。适合内部审计「导出式」翻页；普通控制台用普通 search_after 通常够用。

### 5.6 Scroll API？一般不给前端翻页用

`scroll` 适合**一次性导出**全量，占上下文，不适合大量用户交互翻页。  
对外列表：**search_after（± PIT）**；后台批导：scroll 或 search_after 导出任务。

### 5.7 索引与查询怎么配才快

| 项 | 建议 |
|----|------|
| 过滤 | `userId` + `ts` 放 **filter**（可缓存），不要随意 match 全文 |
| 映射 | `userId` keyword；`ts` date；`logId` keyword |
| 索引 | 按天滚动；查询只命中 `from～to` 覆盖的那些天（**缩小索引集合**） |
| routing | 列表若总按 userId 查，可考虑 `routing=userId` 写入（与点查 routing=logId 可能冲突，二选一或两套索引） |
| _source | 列表只返回摘要字段；点开详情再 Get by id / 回源文件 |
| 上限 | `limit` 封顶（如 50/100）；禁止一次 1000 |

**按天索引缩范围（Node 示意）：**

```js
// from/to → ["logs-2026.10.17", "logs-2026.10.18", "logs-2026.10.19"]
const indices = listDailyIndices(from, to).join(",");
await es.search({ index: indices, body: { ... } });
```

不要每次搜 `logs-*` 全别名除非时间窗很短。

### 5.8 与冷存（大文件）的关系

| 能力 | 数据从哪来 |
|------|------------|
| 按 userId+时间列表 | **ES**（必须事先写入/回填带 userId、ts 的文档） |
| 按 logId 详情 | ES hit 或回源大文件 |

因此：上报或异步管道应**尽量把 userId、ts、logId、摘要写入 ES**；仅「点查回填」不够支撑列表。若历史只有文件没有索引，需要：

- 离线任务按时间扫描冷存建索引，或  
- 产品声明「仅支持近 N 天检索」（与 ILM 对齐）  

### 5.9 API 设计注意点

1. **cursor 不透明**：前端只存字符串，不自己拼 sort  
2. **条件变更必须清空 cursor**（改时间窗 / userId 从第一页来）  
3. **鉴权**：强制 `userId == 当前用户` 或管理员才可查他人  
4. **限流**：列表比点查更伤 ES，Nginx + 应用对 `/api/logs` 查询接口单独限更严  
5. **总数 total**：深分页时 `track_total_hits` 很贵；可只返回 `hasMore`，或总数单独受限查询  

### 5.10 Node.js 列表分页要点

```js
// 示意
async function listLogs({ userId, from, to, limit = 50, cursor }) {
  const pageSize = Math.min(limit, 100);
  const { searchAfter, window } = decodeCursor(cursor) ?? {
    searchAfter: undefined,
    window: firstWindow(from, to), // 例如按天切的第一窗
  };

  const resp = await es.search({
    index: indicesForWindow(window),
    size: pageSize,
    body: {
      query: {
        bool: {
          filter: [
            { term: { userId } },
            { range: { ts: { gte: window.start, lte: window.end } } },
          ],
        },
      },
      sort: [{ ts: "desc" }, { logId: "asc" }],
      search_after: searchAfter,
      _source: ["logId", "ts", "level", "tag", "msgPreview"],
    },
    request_timeout: "3s",
  });

  const hits = resp.hits.hits;
  let next = null;
  if (hits.length === pageSize) {
    next = encodeCursor({
      window,
      searchAfter: hits[hits.length - 1].sort,
    });
  } else {
    const w = nextWindow(window, from, to);
    if (w) next = encodeCursor({ window: w, searchAfter: undefined });
  }
  return { items: hits.map(unwrap), nextCursor: next };
}
```

注意：列表接口同样设超时、连接池；禁止一次拉太大 `_source`。

### 5.11 方案对照（面试表）

| 方案 | 适用 | 不适用 |
|------|------|--------|
| from/size | 前几页 | 深翻页、长跨度 |
| **search_after 游标** | 常规控制台翻页 | —（主推） |
| **时间切片 + 游标** | from/to 很长 | —（主推组合） |
| PIT + search_after | 长会话稳定快照、导出 | 短查询不必 |
| scroll | 后台一次性导出 | 多用户交互翻页 |
| 限制最大时间跨度 | 产品可接受时最省心 | 必须查很久历史时需切片 |

### 5.12 本节收口

> 按 userId+时间捞日志：用 filter + 按天缩索引；分页用 **search_after（ts+logId）** 不透明 cursor；时间太长则 **切时间窗** 或限制跨度；需要稳定快照再上 PIT。列表靠 ES 文档，冷文件不负责「按用户扫描」。

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


