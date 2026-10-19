# 日志冷热架构：大文件 + ES 查询 + Nginx 限流

> **日志列**：发生时间、功能模块、代码文件:行号、附加内容。  
> **ES 主索引维度**：**发生时间 + 功能模块**（列表按此筛）；时间长则分页。  
> **详情**：用 **logId** 取全文（ES → miss 回源大文件并回填）。  
> 后台 **Node.js**。本文含字段模型、查询链路、筛选分页、回填、Nginx、Node 难点。

---

## 1. 架构定位

| 层 | 角色 |
|----|------|
| 大文件系统 | **原文真相源**（含完整附加内容）；按 **logId** 读 |
| Elasticsearch | **检索主入口**：按 **eventTime + module** 过滤分页；可带 file/line 摘要；ILM 过期 |
| API 服务 | **列表 = ES search**；**详情 = ES get，miss 回源文件** |
| Nginx | TLS、路由、列表/详情限流 |

```text
前端控制台
    → Nginx
    → Log Query API（Node.js）
         │
         │ ① 主路径：时间 + 功能模块（+ 可选条件）列表
         ├─ GET /api/logs?from&to&module=checkout&file&line&q&cursor=
         │     → ES：filter(eventTime, module, …) + search_after
         │     → 返回：logId, eventTime, module, file, line, extraPreview…
         │
         │ ② 详情：点开一行
         └─ GET /api/logs/{logId}
               → ES.get → miss 则大文件回源并回填
```

### 1.1 字段模型（列定义）

| 列（业务） | 字段名建议 | 是否进 ES 索引 | 说明 |
|------------|------------|----------------|------|
| 日志发生时间 | `eventTime` | **是（主）** | date；按天滚索引、range 过滤、排序 |
| 具体功能模块 | `module` | **是（主）** | keyword；如 `network` / `checkout` / `pay` |
| 代码文件 | `file` | 存 + 可选筛 | keyword，如 `src/net/http.ts` |
| 具体行数 | `line` | 存 + 可选筛 | integer |
| 附加内容 | `extra` | 摘要 / 不全量索引 | 大段进文件；ES 可留 `extraPreview` |
| 日志 ID | `logId` | `_id` + keyword | 详情与回源主键 |
| 归属用户（可选） | `userId` | 建议有 | 多租户鉴权、强制数据隔离 |

> 「以时间和功能模块做索引」= 查询热路径只靠这两维高效过滤；`file`/`line`/`extra` 是展示与辅筛，**不要指望对巨大 extra 做全量倒排当主查询。**

### 1.2 主从关系

| 能力 | 靠谁 |
|------|------|
| 按时间 + 模块列表 / 分页 | **ES** |
| 辅筛 file / line / 关键字 | **ES**（字段已映射的前提下） |
| 完整附加内容 | **大文件**（或 ES 仅存预览） |
| 单条定位 | **logId** |

原则：列表入口是 **eventTime 范围 + module（常用）**，不是 logId；logId 出详情。

---

## 2. 整体日志查询链路（端到端）

### 2.1 写入（查询的前提）

```text
上报
  → 接入（鉴权、限流、batch 幂等）
  → 大文件：存完整记录（含 extra 全文），key=logId
  → ES 文档（索引热字段）：
       eventTime, module,          ← 主索引维度
       file, line,                 ← 定位代码
       extraPreview, logId, userId, fileUri
```

mapping 要点：

```json
{
  "mappings": {
    "properties": {
      "eventTime": { "type": "date" },
      "module":    { "type": "keyword" },
      "file":      { "type": "keyword" },
      "line":      { "type": "integer" },
      "logId":     { "type": "keyword" },
      "userId":    { "type": "keyword" },
      "extraPreview": { "type": "text", "fields": { "raw": { "type": "keyword", "ignore_above": 256 } } },
      "fileUri":   { "type": "keyword", "index": false }
    }
  }
}
```

- 按 **eventTime** 建日索引：`logs-2026.10.19`  
- 查询常带 `module` → filter 走 keyword，非常快  
- `extra` 全文一般 **不进 ES** 或只 preview，避免索引膨胀  

### 2.2 查询（控制台）

```text
1. 必选：时间范围 from/to（发生时间）
2. 常用：功能模块 module（如 network / checkout）
3. 可选：file、line、附加内容关键字 q、userId（管理员）
4. 查询 → 翻页（cursor）
5. 点行 → logId 详情（全文 extra）
```

### 2.3 详情读路径（logId）

```text
GET /api/logs/{logId}
  → ES.get → 有预览可够用则返回
  → 要全文 / miss → 文件服务（fileUri 或 logId）→ 可回填 ES 摘要
```

并发回填见第 3 节。

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
| 列表：时间 + 模块 | `range(eventTime)` + `term(module)` 进 **filter**；按天缩索引 |
| 点查 logId | `GET _doc/{logId}` |
| `_source` 太大 | 列表不要塞完整 `extra`；只要 preview |
| 深分页 | 禁用大 from；用 search_after（按 eventTime+logId） |
| 对 extra 全文乱搜 | 仅 `extraPreview` match；全文回源文件 |
| 高频单模块 | 可考虑 `routing=module` 或按模块拆索引前缀 |

---

## 5. 主查询：发生时间 + 功能模块 + 分页

### 5.1 整个查询怎么走

```text
必选：发生时间 from/to
常用：功能模块 module（例：network = 网络相关模块）
可选：file、line、userId、附加内容关键字 q
操作：查列表 → cursor 翻页 → 点 logId 看完整 extra
```

API：

```text
GET /api/logs
  ?from=2026-10-01T00:00:00Z
  &to=2026-10-19T00:00:00Z
  &module=network                 # 功能模块（主筛）
  &file=src/net/client.ts         # 可选：代码文件
  &line=128                       # 可选：行号
  &userId=u1                      # 可选/鉴权注入
  &q=timeout                      # 可选：搜 extraPreview
  &limit=50
  &cursor=...
```

示例文档：

```json
{
  "logId": "01JB…",
  "eventTime": "2026-10-18T12:01:02.123Z",
  "module": "network",
  "file": "src/net/http_client.ts",
  "line": 128,
  "extraPreview": "request timeout after 3000ms host=…",
  "userId": "u1",
  "fileUri": "fs://logs/01JB…"
}
```

### 5.2 为什么「时间 + 模块」做索引够用

| 维度 | 作用 |
|------|------|
| **eventTime** | 日索引裁剪 + range 过滤 + 排序翻页，砍掉无关天的数据 |
| **module** | keyword 精确命中「网络 / 支付 / 登录…」，基数远小于全文 |

`file`+`line` 用于定位代码，作**附加 filter**即可，一般不必当路由键。  
`extra` 变化大、体积大 → **文件存全文，ES 只 preview**，避免索引爆炸。

### 5.3 ES 查询体（filter 叠主维 + 辅维）

```json
{
  "size": 50,
  "query": {
    "bool": {
      "filter": [
        { "range": { "eventTime": { "gte": "…", "lte": "…" } } },
        { "term":  { "module": "network" } },
        { "term":  { "file": "src/net/http_client.ts" } },
        { "term":  { "line": 128 } },
        { "term":  { "userId": "u1" } }
      ],
      "must": [
        { "match": { "extraPreview": "timeout" } }
      ]
    }
  },
  "sort": [
    { "eventTime": "desc" },
    { "logId": "asc" }
  ],
  "_source": ["logId", "eventTime", "module", "file", "line", "extraPreview"]
}
```

无 `q` 时去掉 `must`，纯 filter 更快。

### 5.4 分页：search_after（禁深 from/size）

排序键：`eventTime desc` + `logId asc`。

```json
"search_after": [ 1729000000123, "01JB…" ]
```

```text
响应：{ "items": [...], "nextCursor": "…" | null }
下一页：相同 from/to/module/file/… + cursor
```

硬规则：

1. cursor 不透明  
2. **改时间、module、file、line、q 任一条件 → 重置 cursor**  
3. cursor 内带 `filtersHash`，防条件与游标错配  

### 5.5 时间跨度大：按天切窗 + 游标

```text
cursor = { filtersHash, window(start,end), searchAfter }
本窗翻完 → 下一天窗口，searchAfter 清空
```

或产品限制单次最长 1～7 天。有 `module` 强过滤时可略放宽，但仍只打开相关日索引。

### 5.6 按功能模块优化（如网络模块）

| 做法 | 说明 |
|------|------|
| 统一 `module` 枚举 | 上报与前端下拉同一套：`network`/`checkout`/… |
| 同索引 + term | 实现简单，默认推荐 |
| 模块流量极大 | 可选索引前缀 `logs-network-YYYY.MM.DD` 或 `routing=module` |
| 列表字段 | 固定出 eventTime、module、file、line、extraPreview；完整 extra 走详情 |

### 5.7 PIT / scroll

| API | 用途 |
|-----|------|
| search_after | 控制台翻页（主推） |
| PIT + search_after | 导出、长快照 |
| scroll | 仅后台批导 |

### 5.8 索引与性能清单

| 项 | 建议 |
|----|------|
| 日索引 | 按 **eventTime** 命名并只搜 from～to 覆盖天 |
| 主 filter | 必带 eventTime range；尽量带 module |
| routing | 可选 `userId` 或 `module`（团队统一一种） |
| limit | ≤ 50/100 |
| total | 用 nextCursor/hasMore，慎用精确 deep hit count |
| 限流 | 列表严于详情 |

### 5.9 与冷存

| 步骤 | 存哪 |
|------|------|
| 时间+模块列表 | ES |
| file/line/preview | ES |
| 完整附加内容 extra | **大文件** |
| 详情 | logId → ES / 文件 |

### 5.10 Node.js 列表示意

```js
function buildFilters({ from, to, module, file, line, userId }) {
  const filter = [
    { range: { eventTime: { gte: from, lte: to } } },
  ];
  if (module) filter.push({ term: { module } });
  if (file) filter.push({ term: { file } });
  if (line != null) filter.push({ term: { line } });
  if (userId) filter.push({ term: { userId } });
  return filter;
}

async function listLogs(params) {
  const pageSize = Math.min(params.limit ?? 50, 100);
  const filtersHash = hashFilters(params);
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
        filter: buildFilters({
          ...params,
          from: window.start,
          to: window.end,
        }),
        ...(params.q
          ? { must: [{ match: { extraPreview: params.q } }] }
          : {}),
      },
    },
    sort: [{ eventTime: "desc" }, { logId: "asc" }],
    _source: ["logId", "eventTime", "module", "file", "line", "extraPreview"],
  };
  if (searchAfter) body.search_after = searchAfter;

  const resp = await es.search({
    index: indicesForWindow(window), // 按 eventTime 日索引
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

### 5.11 方案对照

| 方案 | 适用 |
|------|------|
| eventTime + module filter + search_after | 主查询（推荐） |
| 时间切片 + 游标 | 跨度很长 |
| file/line 辅 filter | 定位到具体代码行 |
| extra 只 preview | 控索引体积 |
| 按 module 拆索引 | 单模块流量极大时 |

### 5.12 本节收口

> 列 = **发生时间、功能模块、文件:行号、附加内容**。ES **主索引时间 + 模块**；列表 filter + **search_after(eventTime, logId)**；完整 extra 在大文件，详情用 logId 回源。改 module/时间等条件必须重置 cursor。

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
| 同一 logId 详情回填 | 可辅助短缓存 | **互斥 / 幂等（主）** |
| 列表（时间+module±file/line） | **更严**限流 | filter + search_after + 时间切片 |
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
| 列表筛不出某模块 | 上报必须写准 `module`/`eventTime` 进 ES |
| **时间+模块分页** | filter；**search_after(eventTime,logId)**；改条件重置 cursor；切窗 |
| extra 撑爆索引 | 全文进文件，ES 只 `extraPreview` |
| 条件用 match 扫全体 | module/file 用 term；关键字只打 preview |
| 并发 miss 打穿文件服务 | 详情路径互斥回填 + `_id` 幂等 |
| 多用户总流量过大 | Nginx 限流；列表接口更严 |
| 单 logId 被刷 | Nginx 按 `$request_uri` 限流 |
| ES 点查慢 | Get by id；列表控 `_source`、按天缩索引 |
| ES 过期 | ILM N 天与可查窗口对齐 |
| 缓存串权 | 带权限则勿盲目 proxy_cache，或 key 含 userId |
| ES 写失败 | 仍返回文件内容；指标告警 |
| 穿透非法 id | 校验 + 负缓存 |
| **Node 阻塞 loop** | 禁止请求路径重 CPU；大日志截断/Worker |
| **Node 多实例惊群** | Map singleflight + Redis NX 锁 |
| **Node 内存/连接打满** | 回源并发上限 + Agent 连接池 + 超时预算 |

---

## 11. 面试收口

> 日志列：**发生时间、功能模块、文件:行号、附加内容**。ES **以 eventTime + module 为主索引维度**做 filter 列表，并用 **search_after(eventTime, logId)** 分页；跨度大则按天切窗。file/line 作辅筛，extra 全文放文件、ES 只 preview。  
> **logId 详情回源**；写入时维度必须进 ES。Nginx 严限列表；Node 管超时/连接池/singleflight；ILM 对齐可查窗口。

---

## 12. 相关文档

- [02-interview-qa.md](./02-interview-qa.md) — 缓存击穿、限流、网关相关题；场景题 I4  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md) — 弹力与限流  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md) — Cache-Aside  


