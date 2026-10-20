# 日志冷热架构：HDFS 真相源 + 查询回填 ES + Nginx 限流

> **日志列**：发生时间、功能模块、代码文件:行号、附加内容。  
> **真相源**：**HDFS** — 上报只保证写入 HDFS。  
> **HDFS 取数键**：**userId → 该用户最近一次上报的日志包**（一次上报内可含多行）；**无 fileUri、无 logId 回源**。  
> **列表 query**：**userId + 时间范围 + module + 可选筛选** — **不含 logId / fileUri**。  
> **ES**：列表 miss 时，解析该次上报包 → 按条件过滤 → 写入 **瘦文档**，之后列表可走 ES。  
> **全文 extra**：仍在同一次上报包的行内；展开时 **再次按 userId 取最近上报包**，用行字段（eventTime/module/file/line 等）匹配，**不写 ES**。  
> 后台 **Node.js**。

---

## 1. 架构定位

| 层 | 角色 |
|----|------|
| **HDFS** | 真相源；**按 userId 存放上报包**；查询时取 **该用户最近一次上报** |
| Elasticsearch | 列表加速：按 eventTime + module（+ 辅筛）分页；由列表 miss 从「最近上报包」回填瘦文档 |
| API | 列表：ES → miss 则 `hdfs.latestUpload(userId)` → 过滤 → 首屏直出 + 异步 Bulk；全文：再取同包匹配行 |
| Nginx | TLS、路由、列表限流 |

```text
上报：
  客户端一次上报（可含多行完整日志，含 extra）
  → 写入 HDFS：/logs/{userId}/latest  （或 /logs/{userId}/{uploadTs}，查询取最新）
  → ES 不同步写满

查询：
  GET /api/logs?userId&from&to&module&file&line&q&cursor
       │  （无 logId、无 fileUri）
       ├─ ES 已有该 user 该条件段？→ search_after 返回瘦行
       └─ miss → HDFS.getLatestUpload(userId)
                → 流式解析包内每一行
                → 按 from/to/module/… 过滤
                → 首屏直出 + 异步 Bulk 瘦文档进 ES

展开某一行全文：
  → 再次 HDFS.getLatestUpload(userId)（或短缓存该包）
  → 用列表行上的 eventTime+module+file+line(+preview) 匹配到包内行
  → 返回该行完整 extra
  → 不写 ES、无 logId、无 fileUri
```

### 1.1 字段与 API 契约

| 字段 | 列表 query？ | 进 ES？ | 说明 |
|------|--------------|---------|------|
| `userId` | **必有** | keyword | **唯一 HDFS 取数键**（再取「最近一次上报」） |
| `from` / `to` | **必有** | 对应 eventTime range | 时间窗 |
| `module` | 常用 | keyword 主维 | 如 network / route |
| `file` / `line` / `q` | 可选 | 辅筛 / preview match | |
| `cursor` | 翻页 | — | 改条件须重置 |
| `extra` | 否 | **不进** | 只在上报包行内 |
| `extraPreview` | 否（仅响应） | 短文本 | 列表展示 |
| ~~logId~~ / ~~fileUri~~ | **禁止** | — | 不出现在 query，也不做回源键 |

> **HDFS 路径语义**：不是按时间扫全集群，而是 **`userId` → 最近一次上报包**。时间/模块过滤在 **包内行** 上做。

### 1.2 主从关系

| 能力 | 靠谁 |
|------|------|
| 拿到原始数据 | **HDFS：userId → latest upload** |
| 按时间+模块列表/分页 | 包内过滤；热后走 **ES** |
| 完整 extra | **同一上报包内匹配行**（可短缓存整包） |
| 定位某一行 | 列表条件缩小 → 行字段匹配包内行（无 id / fileUri） |

### 1.3 时间 + 模块索引够不够？

**够用做列表主维**（在「已拿到该用户最近上报包」之后）：强制时间窗 + module + 分页。  
extra 高基数 → 不进 ES 主索引。  
无时间只查 module → 不够。

```text
✅ query = userId + from/to + module + 可选筛选
✅ HDFS = getLatestUpload(userId)
✅ ES = 瘦文档；_id 可用 hash(userId,eventTime,module,file,line,preview) 仅幂等
❌ 不要 logId / fileUri 查询或回源
```

---

## 2. 端到端链路

### 2.1 上报与存放

```text
一次上报 batch（多行）
  → HDFS.put(/logs/{userId}/uploads/{uploadId}) 并更新 latest 指针
  → 或直接覆盖 /logs/{userId}/latest
行字段：eventTime, module, file, line, extra, userId
```

「最近一次」= latest 指针指向的那一个包（产品若要「最近 N 次」可扩展，本文默认 **一次**）。

### 2.2 列表 miss → 解析最近上报 → ES

```text
blob = hdfs.getLatestUpload(userId)
for row in streamParse(blob):
  if !match(row, {from,to,module,file,line,q}): continue
  page / Bulk ← toThinDoc(row)   // 无完整 extra
```

瘦文档：

```text
{ eventTime, module, file, line, extraPreview, userId }
_id = hash(userId, eventTime, module, file, line, extraPreview)
```

mapping 不再含 `fileUri` / `logId`。

### 2.3 列表 query 契约

```text
GET /api/logs?userId=u1&from=…&to=…&module=network&file&line&q&limit&cursor

禁止：logId、fileUri
响应 items[]：eventTime, module, file, line, extraPreview, userId
```

### 2.4 全文：再取最近上报包 + 行匹配

```text
expand(userId, rowHint):  // rowHint 来自列表行：eventTime,module,file,line,extraPreview
  blob = cache or hdfs.getLatestUpload(userId)
  row = findLine(blob, rowHint)
  return row.extra
  // 不写 ES
```

若最近又发生新上报，包已变：以 **当前 latest** 为准；匹配不到则提示「日志包已更新，请重新查询列表」。

---

## 3. 并发：同一用户列表 +「最近上报 → ES」

「同一段」= 同一 `userId` + 相近列表条件（尤其同一 module 窗）。

### 3.1 问题

| 现象 | 原因 |
|------|------|
| 读放大 | 同条件反复 ES search |
| 回填惊群 | N 请求同时 `getLatestUpload` + 解析 + Bulk |
| HDFS 打穿 | 同 user 最新包被反复拉取 |
| 包更新竞态 | 查询中用户又上报，latest 指针变化 |

### 3.2 读路径（ES 已热）

- filter：`userId` + `eventTime` + `module` + 辅筛  
- listKey 短缓存 + 查询 singleflight  
- search_after：`eventTime` + `rowKey`（`rowKey=_id` 哈希，**不对前端当查询参数**）

```text
listKey = hash(userId, from, to, module, file, line, q, cursor)
```

### 3.3 写路径（列表 miss：最近上报包 → ES）

**唯一写 ES 路径**：列表 miss → 拉该 user **最新一次上报** → 过滤 → Bulk 瘦文档。  
**不做** logId/fileUri 点级回填。

#### A0. 快返

```text
1. singleflight(userId + uploadId)：整包只拉/解析一次
2. 流式按条件过滤
3. 凑满 page → 立刻返回；命中行异步 Bulk
4. markWarm(userId, uploadId, filtersHash 或时间桶)
5. 翻页：ES 够则走 ES；包未灌完可继续用内存游标
```

```text
blob = getLatestUpload(userId)           // HDFS 只认 userId→最近一次
uploadId = blob.id / etag / mtime
lockKey = "backfill:"+userId+":"+uploadId

if tryLock(lockKey):
  for row in stream(blob):
    if match(filters): enqueueBulk(toThinDoc(row)); pushPage…
  markWarm(...); unlock
else:
  await 同 uploadId 的解析结果 / 首屏缓存
```

#### A1. 换 module（network → route）

同一 `userId`、同一最新包：

| 策略 | 做法 |
|------|------|
| **一包多用** | 第一次拉包时，侧写包内 **所有 module** 的瘦文档进 ES |
| 当前 module 优先凑页 | 不挡首屏 |
| listKey 含 module | 结果页缓存不串 |

同一包只付 **一次 HDFS 读 + 一次解析**；换 module 尽量只打 ES。

#### B. 全文

```text
同 userId 再取 latest（优先用进程/Redis 短缓存整包，TTL 数十秒）
→ 行字段匹配 → 返回 extra → 不写 ES
```

对「同 user 同包」展开做 singleflight，避免重复读 HDFS。

#### C. 不要做

| 不要 | 原因 |
|------|------|
| logId / fileUri 查询或回源 | 契约禁止 |
| 按时间扫全站 HDFS | 本架构是 **按 user 取最近一次上报** |
| 等整包全进 ES 再返回 | 应用 A0 快返 |
| 完整 extra 进 ES | 体积爆炸 |

#### D. 「写 ES 的状态」存在哪？

要存的不是业务日志本身（日志在 HDFS / 瘦文档在 ES），而是 **回填控制面状态**：这包灌过没有、谁在灌、灌到哪。

| 状态 | 存哪 | Key 示例 | TTL / 寿命 | 说明 |
|------|------|----------|------------|------|
| **latest 指针** | **HDFS**（或上报元数据） | `/logs/{userId}/latest` → uploadId | 随上报更新 | 真相：用户最近一次上报是哪包 |
| **回填锁** | **Redis** | `backfill:{userId}:{uploadId}` | 短（30～120s）+ 可续期 | 同包只允许一路解析/Bulk |
| **温标记 warm** | **Redis（主）** | `warm:{userId}:{uploadId}` 或 `warm:{userId}:{uploadId}:all` | 对齐 ES ILM 或略短 | 「此包已灌进 ES」；列表先看它再决定是否拉 HDFS |
| **分 module 温**（可选） | Redis | `warm:{userId}:{uploadId}:{module}` | 同上 | 若未做「一包侧写全 module」时用 |
| **回填进度**（可选） | Redis | `backfill:prog:{userId}:{uploadId}` = offset/行号/状态 | 随任务 | 大包断点续灌；小包可省略 |
| **列表页缓存** | Redis | `list:{listKey}` | 5～30s | 读路径，不是写进度 |
| **整包短缓存** | 进程内存 / Redis | `blob:{userId}:{uploadId}` | 数十秒 | 降 HDFS 重复读；可只缓存元数据+路径 |
| **实际已索引内容** | **ES 文档** | `_id=rowKey` | ILM | 真正的数据状态；warm 只是加速判断的缓存 |
| **回填任务审计**（可选） | MySQL/PG | 任务表 | 长期 | 运维排查；**不**挡请求路径热读 |

推荐默认（够用）：

```text
HDFS：  latest uploadId（用户最近一次上报）
Redis： backfill 锁 + warm:{userId}:{uploadId} +（可选）进度
ES：    瘦文档本身
```

判定「要不要再写 ES」：

```text
uploadId = hdfs.latestUploadId(userId)

if redis.get("warm:"+userId+":"+uploadId):
    → 只读 ES（按列表条件 search）
else if redis.setNX("backfill:"+userId+":"+uploadId):
    → 拉包 → Bulk → set warm → del lock
else:
    → 等 warm / 共享首屏

若 hdfs.latest 的 uploadId 变了（用户又上报）：
    旧 warm 自动失效（key 含 uploadId）；对新 uploadId 重新回填
```

| 不要把状态只放在… | 原因 |
|-------------------|------|
| **只靠 ES count 当 warm** | 每次先打 ES；空洞/延迟难区分「未灌」与「条件无数据」 |
| **只放 Node 进程内存** | 多实例不一致；重启丢锁/丢 warm → 惊群 |
| **写进 HDFS 日志包内** | 污染真相源；并发更新包危险 |
| **warm 不带 uploadId** | 用户新上报后仍以为已热，读到旧索引或空窗 |

### 3.4 收口

> HDFS = **`userId` → 最近一次上报包**（含 latest uploadId）。  
> **写 ES 状态**：锁 / warm / 进度放 **Redis**（key 带 `userId+uploadId`）；数据在 **ES**；latest 在 **HDFS**。  
> 列表 = 包内过滤 + ES 瘦索引；全文 = 同包行匹配；**无 logId / fileUri**。

---

## 4. ES 查询慢（仅列表）

| 场景 | 做法 |
|------|------|
| user + 时间 + 模块 | 全进 filter；缩索引（可按 user 路由或共享日索引 + term userId） |
| `_source` | 瘦字段 only |
| 分页 | search_after(eventTime, rowKey) |
| ~~点查 id / fileUri~~ | 无 |

---

## 5. 主查询与示例

### 5.1 API

```text
GET /api/logs?userId=u1&from=…&to=…&module=network&limit=50&cursor=…
```

响应行：

```json
{
  "eventTime": "2026-10-18T12:01:02.123Z",
  "module": "network",
  "file": "src/net/http_client.ts",
  "line": 128,
  "extraPreview": "request timeout after 3000ms…",
  "userId": "u1"
}
```

### 5.2 ES 查询体（热路径）

```json
{
  "size": 50,
  "query": {
    "bool": {
      "filter": [
        { "term": { "userId": "u1" } },
        { "range": { "eventTime": { "gte": "…", "lte": "…" } } },
        { "term": { "module": "network" } }
      ]
    }
  },
  "sort": [{ "eventTime": "desc" }, { "rowKey": "asc" }],
  "_source": ["eventTime", "module", "file", "line", "extraPreview", "userId"]
}
```

`rowKey` 仅内部排序/幂等，**不是**前端 query 参数。

### 5.3 冷路径伪代码

```js
async function listLogs(params) {
  // params: userId, from, to, module, file, line, q, cursor — 无 logId/fileUri
  if (await isWarm(params)) return esSearch(params);

  return singleflight(`upload:${params.userId}`, async () => {
    const blob = await hdfs.getLatestUpload(params.userId);
    const page = [];
    for await (const row of parseLines(blob)) {
      if (!match(row, params)) continue;
      enqueueBulk(toThinDoc(row)); // async
      if (page.length < pageSize) page.push(toThinDoc(row));
      if (page.length >= pageSize && !needFullWarm) break; // 首屏可先返回，后台继续灌
    }
    return { items: page, nextCursor: ..., source: "hdfs_latest_upload" };
  });
}

async function readExtra(userId, rowHint) {
  const blob = await cachedLatestUpload(userId);
  const row = findInBlob(blob, rowHint);
  return row?.extra ?? null;
}
```

### 5.4 收口

> 取数只认 **userId 的最近一次上报**；筛选认 **时间+模块+辅筛**；全文认 **同包行匹配**。

---

## 6. ES 过期（ILM）

按日或按 `userId` 策略删热索引；再查 → 再拉该 user latest 上报回填。  
注意：ES 过期 ≠ HDFS latest 包过期；包仍以 HDFS 为准。

---

## 7. Nginx

| 诉求 | Nginx | 应用 |
|------|-------|------|
| 多用户列表 | per-user / IP 限流 | listKey 缓存；**per userId 回填锁** |
| 同 user 冷查惊群 | 限流 | `backfill:userId:uploadId` singleflight |
| 全文展开 | content 接口限流 | 整包短缓存 + 行匹配 |

无 `/api/logs/{logId}`，无 fileUri 专用 location。

---

## 8. 时序（三请求同 user 冷列表）

```text
A/B/C 同 userId 查 module=network
  → A 拉 latest 上报包并加锁 → 过滤 → 首屏 + Bulk（可侧写其它 module）
  → B/C 等同一 uploadId 结果 / warm → 其后 ES
换 route：尽量直接 ES（若侧写已完成）
展开行：读缓存包或再 getLatestUpload → 字段匹配 → extra
```

---

## 9. Node 要点

- **singleflight 键**：`userId` 或 `userId+uploadId`（拉包/解析），不是 logId  
- 整包可短缓存（内存/Redis），控制 RSS；超大包流式解析  
- Bulk 异步；首屏不 await 整包进 ES  
- 全文不写 ES  
- 连接池、超时、回填并发上限、event loop lag 同前

---

## 10. 难点汇总

| 难点 | 对策 |
|------|------|
| 无 logId / fileUri | HDFS 只按 **userId→最近一次上报**；行靠字段匹配 |
| 列表筛选 | 包内 filter；热后 ES（时间+module） |
| 换 module | 一包解析，侧写多模块瘦文档 |
| 同 user 惊群 | uploadId 级锁 + 整包短缓存 |
| 上报更新导致包变 | warm 绑定 uploadId；变则重拉；展开匹配失败则提示重查列表 |
| **写 ES 状态存哪** | Redis：`backfill` 锁 + `warm:{userId}:{uploadId}`；latest 在 HDFS；文档在 ES |
| extra 体积 | 不进 ES；仅包内 |

---

## 11. 面试收口

> **列表 query**：`userId + 时间 + module + 筛选`，无 logId、无 fileUri。  
> **HDFS**：`getLatestUpload(userId)` —— 用户最近一次上报的日志包。  
> **ES**：列表 miss 时从该包过滤后 Bulk 瘦文档。  
> **全文**：再取同包，用行字段匹配；不写 ES。

---

## 12. 相关文档

- [02-interview-qa.md](./02-interview-qa.md) — I4  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md)  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md)  
