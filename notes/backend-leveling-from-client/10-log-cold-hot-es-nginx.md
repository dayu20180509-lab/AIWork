# 日志冷热架构：HDFS 真相源 + 查询回填 ES + Nginx 限流

> **日志列**：发生时间、功能模块、代码文件:行号、附加内容。  
> **真相源**：**HDFS（或等价大文件存储）** — 上报只保证写入 HDFS。  
> **ES**：**按需热索引** — 列表查询 miss 时从 HDFS 解析并写入 **瘦文档**，之后列表走 ES。  
> **列表 query 入参**：**userId + 时间范围 + module + 可选筛选** — **不含 logId**。  
> **全文 extra**：**不按 logId 点查/回填**；列表行带回 `fileUri`，展开时直读 HDFS。  
> **ES 主索引维度**：**发生时间 + 功能模块**；时间长则分页。  
> 后台 **Node.js**。

---

## 1. 架构定位

| 层 | 角色 |
|----|------|
| **HDFS / 大文件** | **唯一真相源**（含完整 extra）；按 **时间分区** 存放；行内可解析字段；上报必达 |
| Elasticsearch | **列表加速层**：按 **eventTime + module**（+ 辅筛）过滤分页；**缺则列表路径从 HDFS 回填瘦文档**；ILM 可丢 |
| API 服务 | **列表**先查 ES；miss → 扫 HDFS → 边筛边 Bulk 瘦文档 → 首屏可直出；**全文**只读 `fileUri`→HDFS |
| Nginx | TLS、路由、**列表**限流（抑回填风暴入口） |

```text
上报：只写 HDFS（按日/小时分区；行内含 eventTime/module/file/line/extra/userId + 落盘 fileUri）
      ES 不保证同步写满（靠列表查询回填）

前端控制台
    → Nginx
    → Log Query API（Node.js）
         │
         │ ① 列表（唯一检索入口；query 无 logId）
         ├─ GET /api/logs?userId&from&to&module=network&file&line&q&cursor
         │     → ES 有该段？→ search_after 返回瘦行（含 fileUri）
         │     → ES 没有？→ 从 HDFS 拉时间窗 → 读时过滤 → 首屏直出 + 异步 Bulk 瘦文档
         │
         │ ② 展开全文（不是 logId 详情接口）
         └─ GET /api/logs/content?fileUri=…   （或签发 HDFS/对象存储直链）
               → 鉴权后 HDFS 读该路径全文 extra
               → **不写 ES、不用 logId、不做「点查回填」**
```

### 1.1 字段模型（列定义）

| 列（业务） | 字段名建议 | 列表 query？ | 是否进 ES | 说明 |
|------------|------------|--------------|-----------|------|
| 归属用户 | `userId` | **必有**（鉴权注入或入参） | keyword | 多租户隔离；进 listKey |
| 日志发生时间 | `eventTime` | **必有** from/to | **是（主）** | date；日索引、range、排序 |
| 功能模块 | `module` | **常用** | **是（主）** | keyword；如 `network` / `route` |
| 代码文件 | `file` | 可选辅筛 | keyword | 如 `src/net/http.ts` |
| 行号 | `line` | 可选辅筛 | integer | |
| 附加内容关键字 | `q` | 可选 | 只打 `extraPreview` | 不搜完整 extra |
| 附加内容全文 | `extra` | **否** | **不进** | 只在 HDFS；列表最多 `extraPreview` |
| 文件定位 | `fileUri` | **否**（仅响应） | 存、`index:false` | 展开全文用；**替代旧版 logId 详情键** |
| 翻页 | `cursor` | 翻页时带 | — | 不透明；改条件须重置 |

> **明确不做**：列表/详情 query 带 `logId`；`GET /api/logs/{logId}`；按 logId 回源或回填 ES。

### 1.2 主从关系

| 能力 | 靠谁 |
|------|------|
| 按 userId + 时间 + 模块列表 / 分页 | **ES**（冷则 HDFS 扫窗回填） |
| 辅筛 file / line / 关键字 | **ES** 或扫 HDFS 时内存 filter |
| 完整附加内容 | **HDFS（fileUri）** |
| 「定位到某一行」 | **先列表缩小 → 点行用 fileUri 读全文**（无 logId 捷径） |

### 1.3 判断：只靠「时间 + 功能模块」做索引够不够？

| 字段 | 重复度 | 含义 |
|------|--------|------|
| **module** | **高重复** | 枚举少；靠时间窗 + 分页消化 |
| **extra** | **基本不重复** | 不做列表主索引；全文只在 HDFS |
| **eventTime** | 连续分布 | range + 按天拆索引 |

**够用——作为列表主索引组合**（必须带时间窗）。  
**不够——**若无时间只查 module，或指望用完整 extra / logId 当列表入口。

| 需求 | 时间+module | 还要什么 |
|------|-------------|----------|
| 某段时间某模块列表 | **够** | userId + search_after |
| 再定位文件行 | 起步够 | `file`/`line` 辅筛 |
| 关键字 | 不够单独完成 | `q`→`extraPreview` |
| 看某一条完整 extra | 不靠这两维 | 列表行的 **fileUri → HDFS** |
| 无时间只选 module | **不够** | 产品强制时间范围 |

```text
✅ 列表 query = userId + from/to + module + 可选 file/line/q/cursor
✅ 主索引 = eventTime + module；瘦文档进 ES
✅ 全文 = fileUri → HDFS；不进 ES、不走 logId
❌ 不要：query 带 logId；按 logId 点查/回填
❌ 不要：完整 extra 进 ES 主索引
```

---

## 2. 整体日志查询链路（端到端）

### 2.1 写入分层：HDFS 必达，ES 按需（仅列表驱动）

```text
① 上报 → HDFS 完整行（含 extra）
     路径：/logs/{yyyy}/{MM}/{dd}/{HH}/…（可再按 userId 分区）
     每行可解析：eventTime, module, file, line, extra, userId
     落盘后得到 fileUri（或 path+offset），写入行元数据

② ES（查询触发，只服务列表）
     列表 miss → 扫时间分区 → 解析热字段 → Bulk 瘦文档：
       eventTime, module, file, line, extraPreview, userId, fileUri
     ES _id = hash(fileUri)   // 仅幂等，不对前端暴露为「按 id 查询」
```

mapping（回填时）：

```json
{
  "mappings": {
    "properties": {
      "eventTime": { "type": "date" },
      "module":    { "type": "keyword" },
      "file":      { "type": "keyword" },
      "line":      { "type": "integer" },
      "userId":    { "type": "keyword" },
      "extraPreview": { "type": "text", "fields": { "raw": { "type": "keyword", "ignore_above": 256 } } },
      "fileUri":   { "type": "keyword" }
    }
  }
}
```

- `fileUri` 用于排序 tie-breaker 与展开全文；**不要**再提供 logId 字段给查询 API。  
- 完整 `extra` **永不**进 ES。

### 2.2 列表 query 契约（无 logId）

```text
GET /api/logs
  ?userId=u1                 # 必有（或网关注入，与数据隔离一致）
  &from=…&to=…               # 必有
  &module=network            # 常用
  &file=…&line=…&q=…         # 可选
  &limit=50&cursor=…

禁止：?logId=…
响应 items[]：eventTime, module, file, line, extraPreview, fileUri, userId
         （可有 nextCursor；无 logId）
```

### 2.3 全文：fileUri → HDFS（无 logId 回溯）

```text
用户点开某一行
  → 使用该行 fileUri
  → GET /api/logs/content?fileUri=…  （校验归属 userId）
  → HDFS 读全文 extra → 返回
  → 不查 ES、不写 ES、不出现 logId
```

并发与回填优化见第 3 节（**只有列表段级回填**，没有「按 id 点级回填」）。

---

## 3. 并发：同一段列表查询时，ES 读 +「HDFS→ES 写」

「同一段」= 相同或高度重叠的 **列表条件**：同一 `userId` + `from/to` + `module`（及相近辅筛/翻页）。

ES 写 **只来自列表 miss 回填**，不来自全文展开。

### 3.1 会出什么问题

| 方向 | 现象 |
|------|------|
| **读放大** | 同一列表条件重复 search |
| **回填惊群** | N 路同时扫同一时间窗 HDFS、N 路 Bulk |
| **HDFS 打穿** | 冷段列表扫盘过大；全文 fileUri 被刷 |
| **读写互相踩** | 回填 Bulk/merge 拖慢列表 search |

### 3.2 读路径优化（段已热）

| 优化 | 做法 |
|------|------|
| 查询形态 | filter：`userId` + `eventTime` + `module`（+ 辅筛）；禁深 from/size |
| 缩索引 | 只打开 from/to 覆盖的日索引 |
| 短缓存 | `listKey` → 页结果 5～30s |
| Singleflight | 相同 listKey 共用一次 ES search |
| 分页 | search_after（`eventTime` + `fileUri`） |

```text
listKey = hash(userId, from, to, module, file, line, q, cursor)
```

### 3.3 写路径优化（列表 miss：HDFS → ES 瘦文档）

**定论：ES 写入 = 列表查询发现没有 → 从 HDFS 取数写入瘦文档。**  
上报只保证 HDFS。**全文接口不触发写 ES。**

| 触发 | 说明 |
|------|------|
| **段级回填（唯一写 ES 路径）** | 列表条件对应时间桶未热 → 扫 HDFS → Bulk 瘦文档 |
| ~~点级 logId 回填~~ | **不做** |

#### A0. 快返（首屏不等整段灌完）

```text
1. 分区裁剪：只开 from/to 的日/小时目录
2. 流式读 → 按 userId/module/file/line/q 内存过滤
3. 双通道：
     ├─ 快：凑满 page → 立刻返回（含 fileUri）
     └─ 慢：命中行异步 Bulk 瘦文档（_id=hash(fileUri)）
4. 段锁 / 时间桶锁：同桶只扫一路
5. markWarm → 翻页走 ES
```

```text
onHit(row):
  thin = { eventTime, module, file, line, extraPreview, userId, fileUri }
  // 无 logId；无完整 extra
  enqueueBulk(_id=hash(fileUri), thin)
```

#### A1. 切换 module（network → route）

同时间窗换 module 时，避免二次扫盘：

| 策略 | 做法 |
|------|------|
| **时间桶扫一次、热多模块** | 扫桶时所有 module 瘦文档都 Bulk；当前 module 优先凑页 |
| **侧写** | 其它 module 只进 Bulk，不挡首屏 |
| **兄弟预热** | 返回后异步灌同窗常用 module |
| listKey 含 module | network 与 route **结果页缓存不共用** |

```text
backfill(timeBucket, userId):
  for line in hdfs.stream(bucket):
      if line.userId != userId: continue
      enqueueBulk(toThinDoc(line))          // 任意 module
      if line.module == requested && match(filters):
          pushPage(toThinDoc(line))
  markWarm(bucketKey)
```

#### A. 段级回填控制面

```text
list(userId, from, to, module, …filters):
  if warm or es.hasEnoughHits: return es.search(...)
  if tryLock(segOrBucketKey):
      stream HDFS → filter → 首屏可先 return → 继续 Bulk
      markWarm; unlock
  else:
      wait shared first page / warm；超时则本请求只读 HDFS 一页（不抢写锁）
```

| 优化 | 做法 |
|------|------|
| 段/桶锁 | 同窗只一路扫盘 |
| 分区裁剪 | 不扫「近几天全部」 |
| 读时过滤 | 只有条件命中才 Bulk |
| 瘦文档 | 无完整 extra；`_id=hash(fileUri)` |
| 温标记 | 第二次同窗少扫 HDFS |
| 切窗 | 限制最大查询跨度 |

#### B. 全文（fileUri）— 明确与 ES 解耦

```text
GET /api/logs/content?fileUri=…
  → 鉴权：fileUri 归属当前 userId
  → hdfs.read(fileUri) → 返回 extra
  → 禁止：es.get / es.index / logId
```

| 优化 | 做法 |
|------|------|
| 限流 | 对 `/api/logs/content` 单独限流（按 user / fileUri） |
| 短缓存 | 可选按 `userId+fileUri` 缓存全文数十秒（权限必须在 key 内） |
| Singleflight | 同 fileUri 并发展开合并一次 HDFS 读 |
| 不回填 ES | 全文再热也不进 ES |

#### C. 不要做

| 不要 | 原因 |
|------|------|
| query / 路径带 logId | 与契约冲突 |
| 按 logId 读 HDFS 或写 ES | 「不做详情 logId 回溯」 |
| 等全量进 ES 再筛再返回 | 首屏最慢 |
| 未筛选完整行 Bulk 进 ES | 写放大 |
| 完整 extra 进 ES | 体积与 merge 爆炸 |

### 3.4～3.5 集群与 Nginx（列表向）

- 回填 Bulk 与列表 search 尽量节点隔离；盯 Bulk 429、warm 命中率。  
- Nginx：**不能**合并相同列表条件；只做入口限流。合流 / 段锁在应用层。

### 3.6 收口

> 列表 query = **userId + 时间 + module + 筛选**（无 logId）。  
> 冷段快返：裁剪 → 流式过滤 → 首屏直出 → 异步 Bulk 瘦文档；换 module 优先 **时间桶一次扫、侧写多模块**。  
> 全文 = **fileUri → HDFS**，不写 ES、不用 logId。

---

## 4. ES 查询慢（仅列表）

| 场景 | 做法 |
|------|------|
| 时间 + 模块 | `range(eventTime)` + `term(module)` + `term(userId)` filter；缩日索引 |
| `_source` | 只要瘦字段 + `fileUri`；不要 extra |
| 深分页 | search_after（`eventTime` + `fileUri`） |
| 关键字 | 仅 `extraPreview` match |
| ~~点查 logId~~ | **无此路径** |

---

## 5. 主查询：userId + 发生时间 + 功能模块 + 分页

### 5.1 怎么走

```text
必选：userId、from/to
常用：module
可选：file、line、q
操作：查列表 → cursor 翻页 → 点行用 fileUri 看完整 extra
```

```text
GET /api/logs?userId=u1&from=…&to=…&module=network&file=…&line=…&q=…&limit=50&cursor=…
```

响应行示例：

```json
{
  "eventTime": "2026-10-18T12:01:02.123Z",
  "module": "network",
  "file": "src/net/http_client.ts",
  "line": 128,
  "extraPreview": "request timeout after 3000ms host=…",
  "userId": "u1",
  "fileUri": "hdfs://logs/2026/10/18/12/part-000#offset=12345"
}
```

### 5.2～5.3 ES 查询体

```json
{
  "size": 50,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "userId": "u1" } },
        { "range": { "eventTime": { "gte": "…", "lte": "…" } } },
        { "term":  { "module": "network" } }
      ]
    }
  },
  "sort": [
    { "eventTime": "desc" },
    { "fileUri": "asc" }
  ],
  "_source": ["eventTime", "module", "file", "line", "extraPreview", "userId", "fileUri"]
}
```

有 `q` 时加 `must: match extraPreview`。排序 / search_after 用 **eventTime + fileUri**（不用 logId）。

### 5.4 分页规则

1. cursor 不透明；内含 `filtersHash`  
2. 改 userId/时间/module/file/line/q → **重置 cursor**  
3. 大跨度按天切窗  

### 5.5 与冷存

| 步骤 | 存哪 |
|------|------|
| 上报 | **HDFS** |
| 列表 | **ES** 瘦文档（无则列表路径回填） |
| 全文 extra | **HDFS via fileUri** |

### 5.6 Node 列表示意

```js
function buildFilters({ userId, from, to, module, file, line }) {
  const filter = [
    { term: { userId } },
    { range: { eventTime: { gte: from, lte: to } } },
  ];
  if (module) filter.push({ term: { module } });
  if (file) filter.push({ term: { file } });
  if (line != null) filter.push({ term: { line } });
  return filter;
}

async function listLogs(params) {
  // params 不得包含 logId
  const body = {
    size: Math.min(params.limit ?? 50, 100),
    query: {
      bool: {
        filter: buildFilters(params),
        ...(params.q ? { must: [{ match: { extraPreview: params.q } }] } : {}),
      },
    },
    sort: [{ eventTime: "desc" }, { fileUri: "asc" }],
    _source: ["eventTime", "module", "file", "line", "extraPreview", "userId", "fileUri"],
  };
  // … search_after / cursor 同前，tie-breaker 为 fileUri
}
```

### 5.7 收口

> 列表只认 **userId + 时间 + module + 辅筛**。全文只认 **fileUri → HDFS**。全程 **无 logId 查询、无 logId 详情回溯**。

---

## 6. ES 过期（ILM）

按日索引 + ILM 删热层；再查同一段 → 再次列表驱动 HDFS→ES 回填。  
N 天按命中率与磁盘调（常见 3～14 天）。

---

## 7. Nginx：多用户列表 & 全文 fileUri

| 诉求 | Nginx | 应用 |
|------|-------|------|
| 多用户列表总流量 | per-IP / per-user 限流 | listKey 缓存 + 段锁回填 |
| 同一列表冷段 | 限流压入口 | 时间桶锁 + Bulk |
| 同一 fileUri 展开 | 可按 URI/参数限流 | fileUri singleflight；**不写 ES** |
| 鉴权 | 粗拦 | userId 与 fileUri 归属校验（主） |

列表示例（思路同前）：`limit_req` + 短超时 + keepalive。  
全文：`location /api/logs/content` 单独限流；cache key 若开启必须含 `userId`。

**不要**再配置 `/api/logs/{logId}` 专用限流——该接口不存在。

---

## 8. 端到端时序（三用户同冷段列表）

```text
UserA/B/C 同 userId 窗 + module=network，ES 未热
  → Nginx 限流通过
  → A 拿到 bucket 锁 → 流式扫 HDFS → 首屏直出 + 异步 Bulk
  → B/C 等 warm / 共享首屏缓存 → 之后走 ES
展开某一行：
  → content?fileUri=… → HDFS（可 singleflight）→ 不写 ES
```

---

## 9. Node.js 难点（对齐本契约）

### 9.1 事件循环 / 连接池 / 大 body

同前：禁止请求路径重 CPU；ES/HDFS 客户端显式连接池；全文流式读；回源并发上限（p-limit）。

### 9.2 列表回填 singleflight（按 seg/bucket，不按 logId）

```js
const inflight = new Map(); // key = bucketKey or listKey

async function ensureWarm(bucketKey, scanFn) {
  if (inflight.has(bucketKey)) return inflight.get(bucketKey);
  const p = (async () => {
    const locked = await redis.set(`backfill:${bucketKey}`, "1", "EX", 120, "NX");
    try {
      if (!locked) {
        await waitWarm(bucketKey);
        return;
      }
      await scanFn(); // 流式 HDFS → Bulk 瘦文档
      await markWarm(bucketKey);
    } finally {
      inflight.delete(bucketKey);
      if (locked) await redis.del(`backfill:${bucketKey}`);
    }
  })();
  inflight.set(bucketKey, p);
  return p;
}
```

### 9.3 全文 fileUri（不回填 ES）

```js
async function readExtra(userId, fileUri) {
  assertOwned(userId, fileUri);
  return singleflight(`content:${fileUri}`, () => hdfs.read(fileUri));
  // 无 es.get / es.index
}
```

### 9.4 超时预算示例

```text
Nginx read 5s
  └─ 列表：ES ≤200ms；冷段首屏以 HDFS 流式为准，Bulk 异步
  └─ 全文：HDFS ≤2s
```

### 9.5 收口

> 列表用 **bucket/listKey 锁** 防扫盘惊群；全文用 **fileUri 锁** 只护 HDFS 读。**没有 logId 回填链路。**

---

## 10. 难点汇总

| 难点 | 对策 |
|------|------|
| 列表条件无 logId | **本来就不该有**；靠 userId+时间+module+辅筛 |
| 时间+模块分页 | filter + search_after(eventTime, **fileUri**) |
| 换 module 二次扫盘 | 时间桶一次扫、侧写多模块瘦文档 |
| extra 撑爆索引 | 全文只在 HDFS；ES 仅 preview |
| 看完整内容 | **fileUri → HDFS**；不做 logId 详情 |
| 同一段列表并发 | listKey 缓存 + 查询/回填 singleflight |
| ES 写入 | 仅列表 miss；瘦文档 + `_id=hash(fileUri)` |
| 多用户流量 | Nginx + 应用限流；列表更严 |
| 同 fileUri 被刷 | content 接口限流 + singleflight |
| Node | 连接池、超时、回填并发上限、event loop lag |

---

## 11. 面试收口

> **列表 query**：`userId + from/to + module + 可选筛选`，**没有 logId**。  
> **ES**：列表 miss 时从 HDFS 回填 **瘦文档**；换 module 用时间桶一次扫优化。  
> **全文**：**fileUri → HDFS**，**不做 logId 详情回溯，也不因展开全文写 ES**。  
> 快返：读时过滤 + 首屏直出 + 异步 Bulk。Nginx 限流，合流在应用层。

---

## 12. 相关文档

- [02-interview-qa.md](./02-interview-qa.md) — 场景题 I4  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md)  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md)  
