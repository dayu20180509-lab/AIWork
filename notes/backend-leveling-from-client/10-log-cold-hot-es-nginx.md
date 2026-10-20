# 日志冷热架构：HDFS 真相源 + 查询回填 ES + Nginx 限流

> **日志列**：发生时间、功能模块、代码文件:行号、附加内容。  
> **真相源**：**HDFS（或等价大文件存储）** — 上报只保证写入 HDFS。  
> **ES**：**按需热索引** — 用户查询时若 ES 没有，再从 HDFS 解析并写入 ES，之后列表/点查走 ES。  
> **ES 主索引维度**：**发生时间 + 功能模块**；时间长则分页。  
> 后台 **Node.js**。本文含字段模型、查询链路、筛选分页、**查询触发回填**、Nginx、Node 难点。

---

## 1. 架构定位

| 层 | 角色 |
|----|------|
| **HDFS / 大文件** | **唯一真相源**（含完整 extra）；按时间分区与 logId 可读；上报必达 |
| Elasticsearch | **查询加速层**：按 **eventTime + module** 过滤分页；**缺则查询路径从 HDFS 回填**；ILM 可丢 |
| API 服务 | 列表/详情先查 ES；**miss → 读 HDFS → 写 ES（瘦文档）→ 再查或直接返回** |
| Nginx | TLS、路由、列表/详情限流（抑回填风暴入口） |

```text
上报（写路径主）：只写 HDFS（按日/小时分区 + logId）
                    ES 不保证同步写满（可完全不写，靠查询回填）

前端控制台
    → Nginx
    → Log Query API（Node.js）
         │
         │ ① 列表：时间 + 功能模块（+ 可选）
         ├─ GET /api/logs?from&to&module=checkout&…
         │     → ES 有该段？→ search_after 返回
         │     → ES 没有 / 明显空洞？→ 从 HDFS 拉该段 → Bulk 写 ES → 再 search
         │
         │ ② 详情：点开一行
         └─ GET /api/logs/{logId}
               → ES.get 命中 → 返回（preview / 元数据）
               → miss → HDFS.get(logId) →（可选）写瘦文档进 ES → 返回全文
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

### 1.3 判断：只靠「时间 + 功能模块」做索引够不够？

前提（与真实数据形态一致）：

| 字段 | 重复度 | 含义 |
|------|--------|------|
| **module（功能模块）** | **高重复** | 枚举少，如 network/pay/login，大量日志落在同一模块 |
| **extra（附加内容）** | **基本不重复** | 每条日志文案/上下文都不同，基数极高 |
| **eventTime** | 连续分布 | 适合 range + 按天拆索引 |

#### 结论（先说清楚）

**够用——作为列表检索的主索引组合；不够——若指望「不靠时间窗、只靠模块」或「靠 extra 建主索引精确命中一条」。**

更准确的说法：

1. **主查询路径：时间范围 + module + 分页 → 够用，且应该这么做。**  
2. **不要把高重复的 module 当成唯一检索键**（没有时间会扫爆）。  
3. **不要把几乎不重复的 extra 当成列表主索引**（索引体积与写入成本爆炸，且列表场景通常先按模块/时间浏览）。

#### 为什么 module 高重复仍然要索引？

- 作用不是「区分每一条」，而是 **快速砍掉其它模块**（filter 选择性来自「排除」）。  
- 例如一天 1000 万条，network 占 20% → 带 `module=network` 后只扫约 200 万，再分页；不带 module 则更痛。  
- keyword + filter 对低基数字段非常合适，内存友好。

高重复带来的限制：

- 单靠 module **无法**定位到一条日志 → 必须配合 **时间范围** 和 **分页（search_after）**。  
- 时间窗过大 + 热模块 → 命中仍很多 → 用切窗 / 限制最大查询跨度，而不是给 module 加「更复杂索引」硬扛。

#### 为什么 extra 不重复反而更不该做主索引？

| 若对 extra 全文建主索引 | 后果 |
|-------------------------|------|
| 词项几乎条条不同 | 倒排表巨大，磁盘与堆压力大 |
| 写入每条都要更新大量 term | 吞吐下降、合并压力大 |
| 列表交互很少是「先输入整段 extra」 | 投入与用法不匹配 |

正确用法：

- **全文 extra → 大文件**（真相）  
- ES 最多 **`extraPreview` 短文本**（可选 match），用于「超时/报错关键字」辅搜  
- 精确一条 → 用 **logId**（点查），不是靠 extra 唯一值当路由

#### 「时间 + 模块」够用的完整体检表

| 需求 | 时间+module 是否够 | 还要什么 |
|------|-------------------|----------|
| 按模块看某段时间的日志列表 | **够** | search_after 分页 |
| 时间跨度很长 | **基本够** | 按天切窗 / 限跨度 |
| 再定位到某文件某行 | 主维仍够起步 | 加 `file`/`line` **辅 filter**（不必当主索引） |
| 搜附加内容里的某个词 | 主维不够单独完成 | `extraPreview` match，或详情里查文件 |
| 打开某一条完整内容 | 不靠这两维 | **logId → 文件** |
| 无时间条件只选 module | **不够** | 产品强制时间范围 |

#### 架构决策（写入文档的定论）

```text
✅ 主索引维度：eventTime（日索引 + range）+ module（keyword filter）
✅ 列表：必须带时间窗；模块高重复靠分页消化
✅ extra 高基数字段：不进主索引；文件存全文，ES 最多短 preview
✅ file/line：可选辅筛，增强定位，不是替代时间+模块
❌ 不要：无时间只查 module
❌ 不要：对完整 extra 建重度全文主索引当列表入口
```

一句话：**模块越重复，越要靠时间裁剪 + 分页；附加内容越不重复，越不该拿它当列表索引，而应放文件用 logId 取。**

---

## 2. 整体日志查询链路（端到端）

### 2.1 写入分层：HDFS 必达，ES 按需

```text
① 上报（同步承诺只到 HDFS）
  → 接入（鉴权、限流、batch 幂等）
  → HDFS：完整记录（含 extra 全文）
       路径建议：/logs/{yyyy}/{MM}/{dd}/{HH}/… 或按 userId 再分区
       行内可解析出：eventTime, module, file, line, extra, logId, userId

② ES 写入（本架构主方向 = 查询触发）
  → 用户列表/详情 miss
  → 从 HDFS 读该段或该 logId
  → 解析出热字段，Bulk/单条写入 ES（瘦文档）：
       eventTime, module,          ← 主索引维度
       file, line,                 ← 定位代码
       extraPreview, logId, userId, fileUri（HDFS 路径）
```

> **定论**：HDFS 丢了不可恢复；ES 丢了（ILM/宕机）只要 HDFS 在，下次查询再回填即可。  
> 可选增强：上报后异步预热热时段进 ES——**不是本架构必选项**，优化重点仍在「查询 miss → HDFS → ES」。

mapping 要点（回填时按此写）：

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

同一段日志的并发读写与 **HDFS→ES 回填** 优化见第 3 节。

---

## 3. 并发：同一段查询时，ES 读 +「HDFS→ES 写」怎么优化

「同一段」指：相同或高度重叠的查询条件，例如同一 `from/to` + 同一 `module=network`（再加相同翻页附近）。

本架构下 **ES 的写几乎都来自查询**：miss 后从 **HDFS 读出 → 解析 → 写入 ES**。多用户同时查冷段时，会同时触发 **读 ES** 与 **回填写 ES**。

### 3.1 会出什么问题

| 方向 | 现象 |
|------|------|
| **读放大** | 同一条件重复 search；冷段未回填时反复空查 |
| **回填惊群** | N 个请求发现同一段 miss，N 路同时扫 HDFS、N 路 Bulk 写同一热索引 |
| **HDFS 打穿** | 列表 miss 扫分区 + 详情 miss 点读，文件集群带宽/NameNode 压力暴涨 |
| **读写互相踩** | 回填 Bulk/refresh/merge 高峰拖慢正在进行的列表 search |
| **连接与 Node** | 无回填并发闸、无连接池上限，打爆 HDFS、ES 与 event loop |

### 3.2 读路径优化（同一段列表查询）

目标：同样条件的并发读，尽量 **少打 ES、打也要轻**（段已回填之后）。

| 优化 | 做法 |
|------|------|
| **查询形态固定** | 只用 filter（eventTime + module）；禁深 from/size；`_source` 只要摘要字段 |
| **缩索引范围** | 同一段只打开覆盖的日索引，勿 `logs-*` |
| **短结果缓存** | 对「条件哈希 + cursor」缓存列表页 5～30s（Redis）；同一段多人共享一页结果 |
| **Singleflight（查询级）** | 进程内：相同 `queryKey` 共用一个进行中的 Promise；多实例可用 Redis 锁「只放行 1 个打 ES，其它读缓存」 |
| **filter 缓存友好** | 条件稳定、走 filter context，利于 ES 内部缓存 |
| **限流与隔离** | Nginx/应用对列表接口单独限流；回填写与查询可分节点 |
| **分页要浅要稳** | search_after；大跨度切窗，避免一次扫很长一段 |

`queryKey` 示例：

```text
hash(from, to, module, file, line, userId, q, cursor)
```

示意（**段已在 ES 时**）：

```text
list(queryKey):
  if redis.get(queryKey) → return
  if singleflight(queryKey) 已有人在查 → await 同一结果
  else:
      hits = es.search(...)
      redis.setex(queryKey, 15s, hits)
      return hits
```

注意：有权限隔离时，cache key **必须带 userId/角色**，防串数据。  
若判定「段未热」→ 先走 **3.3 段级回填**，再执行上述 search。

### 3.3 写路径优化（查询 miss：HDFS → ES）

**定论：ES 写入的主方向 = 用户查询发现没有 → 从 HDFS 取数写入。**  
不是上报同步双写；上报只保证 HDFS。

两类触发：

| 触发 | 何时 | 从 HDFS 取什么 | 写入 ES |
|------|------|----------------|---------|
| **A. 段级回填** | 列表查询：该 `from/to`+`module` 在 ES 几乎无命中 / 标记未热 | 该时间分区内匹配 module 的一批行 | **Bulk** 瘦文档 |
| **B. 点级回填** | 详情：`ES.get(logId)` miss | 单条 `logId` 全文 | 单条或并入小 Bulk；`_id=logId` |

```text
查询
  → ES 命中？→ 只读，不写
  → miss？→ 读 HDFS → 边筛边写 ES → 首屏尽快回客户端（见下「快返」）
```

#### A0. 快返主路径（首屏不等「整段灌完 ES」）

场景：用户带条件查某一段（`from/to` + `module` + 可选 file/line/q）；HDFS 上是 **近几天完整日志**（未按 module 拆好）；ES 还是冷的。

**错误做法（慢）**：扫完近几天全部 → 全量写入 ES → refresh → 再 search 过滤 → 才返回。首屏 RT = 全量扫盘 + 全量索引。

**正确做法（快）**：

```text
1. 分区裁剪：只打开 from/to 覆盖的日/小时目录（近几天里只碰相关桶）
2. 流式读 HDFS → 内存里按查询条件 filter（module / file / line / q）
3. 双通道并行：
     ├─ 快通道：凑满 pageSize 条「已筛选」瘦结果 → 立刻返回客户端（本页不经过 ES）
     └─ 慢通道：命中行持续 Bulk 进 ES；未命中行直接丢弃（不写 ES）
4. 段锁保证同一 segKey 只有一路在扫；其它请求等 warm 或共享同一路产出的首屏缓存
5. 首屏返回后：后台继续灌完该桶匹配行 → markWarm → 翻页走 ES（3.2）
```

时序示意：

```text
Client                API                     HDFS              ES
  |-- list(filter) -->|
  |                   |-- open day/hour dirs ->|
  |                   |<-- stream lines -------|
  |                   |  parse + filter(module…)
  |                   |  page[] += hit
  |                   |  bulkQueue += thinDoc ----bulk-->|  (异步，不挡)
  |                   |  page满？── return page ──>|
  |<== 首屏 JSON =====|  (此时 ES 可能只写了一部分)
  |                   |  continue scan+bulk… ----------->|
  |                   |  markWarm(segKey)
  |-- page2 --------->|  es.search(search_after) ------->|
  |<== 次页从 ES =====|
```

| 环节 | 怎么快 |
|------|--------|
| **HDFS 读** | 按 `eventTime` 分区目录裁剪；按排序方向优先扫「最新桶」（列表常倒序）；并行打开少量文件，但全局 seg 仍单飞 |
| **筛选** | **读时过滤**：完整日志里只有命中 `module` 等条件的行才进入 page / Bulk；不要先全量进 ES 再筛 |
| **写 ES** | 只写命中行的 **瘦文档**；500～2000 条一刷；`_id=logId`；灌的过程中 **不 await 整段结束** |
| **回传客户端** | **首屏 = 内存里筛好的 page**，路径是 HDFS→API→Client；禁止「等整段 Bulk + refresh 再 search」挡首屏 |
| **次页及以后** | 若 `warm` 已亮或 ES 已够命中 → 走 ES；若仍在灌 → 可继续 HDFS 游标翻页，或短等 refresh |

首屏响应形状（概念）：

```json
{
  "items": [ { "eventTime", "module", "file", "line", "extraPreview", "logId" } ],
  "nextCursor": "…",
  "source": "hdfs_stream",
  "backfill": { "segKey": "…", "status": "running" }
}
```

`source=hdfs_stream` 时：本页排序依赖 **HDFS 文件内时间序 + 扫桶顺序**（需与产品约定一致，通常新→旧按小时桶倒序扫）。  
`source=es` 时：排序与 search_after 由 ES 保证；warm 之后应尽量切到 ES，避免两套分页语义长期并存。

**快写 ES 的要点（仅写筛选命中）**：

```text
onHit(row):
  thin = { eventTime, module, file, line, extraPreview, logId, fileUri }  // 无完整 extra
  bulkBuffer.push(thin)
  if bulkBuffer.length >= N:
      es.bulk(bulkBuffer)   // fire-and-forget / 有界队列，失败重试+幂等
      bulkBuffer.clear()

// 不要：
//   - 把近几天未筛选的完整行 bulk 进 ES
//   - 首屏 await es.refresh() + es.search()
```

若产品强依赖「首屏也必须走 ES」（少见）：至少 **先 Bulk 当前页命中 → `refresh` 该索引 → search 仅这一页条件**，仍不要等整段灌完；但 RT 通常差于「内存直出首屏」。

#### A. 列表段级回填（同一段最容易惊群）

多人同时打开同一时间窗 + 同一 module，且 ES 还是冷的。默认走 **A0 快返**；下面是合流与灌完后的完整控制面：

```text
list(from, to, module, …filters):
  if segmentWarm(...) or es.hasEnoughHits(...):
      return es.search(...)                 // 走 3.2，快

  if tryLock("backfill:seg:"+segKey):
      // A0：流式扫裁剪后的分区
      for line in hdfs.stream(dirs=prune(from,to), order=desc):
          if !match(line, filters): continue
          pushPageOrCursor(line)            // 凑满 page → 可先 return（同 seg 共享）
          enqueueBulk(toThinDoc(line))      // 异步写 ES
      flushBulk(); optional refresh
      markWarm(segKey)
      unlock
  else:
      // 别人正在灌：等首屏缓存 / warm；超时则本请求自己扫一页（只读 HDFS，不抢写锁）
      return waitSharedFirstPage() or hdfsScanOnePage(filters)
```

| 优化 | 做法 | 为什么 |
|------|------|--------|
| **段锁 / Singleflight** | 同一 `segKey` 全局只跑 1 个 HDFS→ES 任务 | 避免 N 路重复扫盘、重复 Bulk |
| **分区裁剪** | HDFS 虽有近几天完整日志，只打开 `from/to` 覆盖的日/小时路径 | 扫盘量从「近几天全部」降到「查询窗」 |
| **读时过滤再写** | 完整行在 API 侧 filter；**只有命中才 Bulk** | ES 只热「用户要的那段条件」，写得快、索引小 |
| **首屏直出** | page 满即返回；不等整段 Bulk/refresh | 快返用户 |
| **按时间桶切段** | 不要一次回填「7 天×全量」；按 **小时/天桶** + module 灌 | 控制单次扫描与 Bulk 体积 |
| **流式 + 小 Bulk** | HDFS 流式读 → 500～2000 条一刷 ES | 控 Node 内存；比逐条 index 快一个数量级 |
| **瘦文档** | 只写 eventTime/module/file/line/extraPreview/logId/fileUri；**完整 extra 留 HDFS** | 回填写放大小，列表够用 |
| **幂等 `_id=logId`** | `create` 或 index 覆盖 | 两路回填重叠、重试均安全 |
| **回填与查询节奏** | Bulk 异步；段灌完或翻页需要时再 `refresh` | 减少 merge 抖动，不挡首屏 |
| **背压** | ES 429 / HDFS 慢 → 暂停 Bulk；全局 **回填并发上限** | 保护双集群；首屏仍可只靠 HDFS 页 |
| **温标记** | Redis：`warm:{segKey}`；或 ES count 探测 | 第二次列表不再扫 HDFS |
| **切窗强迫** | 产品限制最大查询跨度；跨度大先切窗再逐窗回填 | 防止一次 miss 引爆全量扫描 |

`segKey` 示例：

```text
segKey = hash(dayOrHourBucket, module [, userId])
// 列表 from/to 跨多桶时：按桶拆分回填，不要合成一个巨大 segKey
```

#### B. 详情点级回填（logId miss）

```text
GET /api/logs/{logId}:
  doc = es.get(logId)
  if doc: return doc (+ 若要全文且只有 preview → 仍可读 HDFS，不必再写 ES)

  // miss：从 HDFS 取真相
  if tryLock("backfill:id:"+logId):
      body = hdfs.get(logId)          // 失败 → 404/502，勿写 ES
      thin = toThinDoc(body)
      // 推荐：先返回全文给用户，ES 写入异步（不挡 RT）
      enqueue or es.index(_id=logId, thin)
      unlock
      return body
  else:
      // 别人正在回填：可读 HDFS 直接返回，或短等 ES
      return hdfs.get(logId)
```

| 优化 | 做法 |
|------|------|
| **logId 锁** | 同 id 只回源写一次 |
| **先响应用户，再写 ES** | 详情 RT 绑 HDFS；ES 失败不影响返回 |
| **能不写就不写** | 列表已有摘要且用户只要全文 → **只读 HDFS，不写 ES**（除非希望下次点查走 ES） |
| **点级限速** | 令牌桶限制「HDFS→ES index」QPS，避免整页连点打穿 |
| **与段回填去重** | 段 Bulk 已写入的 logId，详情不应再写；`create` 遇冲突即停 |

#### C. 回填写 ES 的集群侧要点（专指 HDFS→ES）

| 项 | 建议 |
|----|------|
| Bulk 大小 | 按文档瘦度调；盯 `indexing` 队列与 429 |
| refresh | 回填中稀疏 refresh；段完成或首屏需要时再 refresh |
| 日索引 | 写入目标 = 该行 `eventTime` 对应日索引，勿写错天 |
| 与读隔离 | 条件允许时：回填 Bulk 打写入节点；用户 search 打查询节点 |
| 观测 | `backfill_seg_QPS`、`backfill_id_QPS`、HDFS 读延迟、ES Bulk 拒绝率、`warm` 命中率、回填锁等待时长 |

#### D. 明确不做什么

| 不要 | 原因 |
|------|------|
| 上报同步强依赖写 ES | 与「查询才写」模型冲突；ES 故障会拖垮上报 |
| **等近几天全量进 ES 再筛再返回** | 首屏最慢路径；应读时过滤 + 首屏直出 |
| 把未筛选的完整日志 Bulk 进 ES | 写放大；用户只要条件命中子集 |
| miss 后逐条同步 index 且无锁 | 惊群写爆 ES |
| 回填完整 extra 进 ES | 体积与 merge 爆炸；全文在 HDFS |
| 一次列表 miss 扫光数周 HDFS | 无切桶 / 无分区裁剪 = 自造全表扫描 |

### 3.4 读写一体时的集群侧优化

| 项 | 建议 |
|----|------|
| 热索引 | 同一时段+热模块被回填时写入集中，分片数别过碎或过粗 |
| 副本 | 读多（段已热）可适当加副本；回填高峰评估写压力 |
| 熔断 | HDFS/ES 慢或拒绝 → 快速失败/降级扫 HDFS 一页，避免重试风暴 |
| 观测 | 查询 QPS/P99、回填 Bulk 拒绝率、refresh/merge、queryKey 与 warm 命中率 |

### 3.5 Nginx 在「同一段」并发里做什么

标准 Nginx **不能**合并「相同列表条件」，也 **不能**替你做 HDFS→ES 回填合流。

| 诉求 | Nginx | 应用 |
|------|-------|------|
| 多用户打同一冷段列表 | 按 IP/用户限流，压入口 | **segKey 锁 + Bulk 回填** + queryKey 缓存 |
| 同一 logId 详情毛刺 | URI 限流、可选短 proxy_cache | **logId 锁** + 幂等写 + 可只读 HDFS |
| 保护 HDFS/ES | 总入口限流、超时 | 回填并发上限、Bulk 背压 |

详见第 7 节。

### 3.6 本节收口

> **ES 写 = 查询 miss 后从 HDFS 回填。**  
> 列表冷段 **快返**：分区裁剪 → 流式读完整日志 → **读时按条件过滤** → **首屏内存直出**；命中行 **异步 Bulk 瘦文档** 进 ES → 温标记后翻页走 ES。  
> 详情 miss：HDFS 取全文 → **先返回** → 可选异步写 ES（logId 幂等+锁）。  
> 读侧（段已热）：瘦查询 + 短缓存 + 查询 singleflight。Nginx 只管限流，合流与回填在应用层。

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

### 5.2 为什么「时间 + 模块」做索引够用（呼应 1.3）

| 维度 | 重复度 | 作用 |
|------|--------|------|
| **eventTime** | 连续分布 | 日索引裁剪 + range + 排序翻页 |
| **module** | **高重复** | keyword 快速排除其它模块；不能单独当唯一键 |
| **extra** | **几乎不重复** | **不做列表主索引**；全文在文件，ES 最多 preview |

`file`+`line`：辅 filter，定位代码，不替代主维。  
定论见 **§1.3**：高重复 module + 时间窗 + 分页 = 列表够用；高基数 extra 进主索引 = 亏本。

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

### 5.9 与冷存（HDFS）

| 步骤 | 存哪 |
|------|------|
| 上报落盘（必达） | **HDFS** |
| 时间+模块列表 | **ES**（无则查询时从 HDFS 段级回填） |
| file/line/preview | ES（回填时写入） |
| 完整附加内容 extra | **HDFS** |
| 详情 | logId → ES miss 则 **HDFS**（可再异步写 ES） |

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

> 列 = **发生时间、功能模块、文件:行号、附加内容**。module **高重复**、extra **低重复** ⇒ 主索引仍是 **时间+模块**（够用做列表），靠时间窗+分页消化重复；**禁止**用完整 extra 当列表主索引。file/line 辅筛；详情 logId 回源。

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

ILM 删掉热索引后：再查同一段 → 再次 **HDFS→ES 回填**，形成闭环。  
列表可查范围可大于 ES 保留天数（更老的靠回填），或产品提示「近 N 天已预热更快」。

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
| 列表筛不出某模块 | HDFS 行内必须能解析准 `module`/`eventTime`；回填写入 ES 时带上 |
| **时间+模块分页** | filter；**search_after(eventTime,logId)**；改条件重置 cursor；切窗 |
| extra 撑爆索引 | 全文留 HDFS，ES 只 `extraPreview`（extra 越不重复越不该主索引） |
| 只选 module 不选时间 | **不够**；产品强制时间范围（module 高重复） |
| 条件用 match 扫全体 | module/file 用 term；关键字只打 preview |
| **同一段日志并发读** | queryKey 短缓存 + 查询 singleflight；瘦 filter；缩日索引 |
| **ES 写入（查询触发）** | miss → HDFS；**段锁 + 流式 Bulk 瘦文档 + 温标记 + 回填限速** |
| 详情 miss 打穿 HDFS/ES | logId 锁 + `_id` 幂等；**先返回 HDFS，ES 可异步写** |
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

> **时间+模块做主索引够不够？** module 高重复 → 必须带时间窗 + 分页；extra 不重复 → 不做列表主索引。  
> **ES 怎么来的？** 查询 miss → 从 **HDFS 回填**（段级 Bulk / 点级幂等）；上报只保证 HDFS。  
> **同一段并发**：读用缓存+查询 singleflight；写用 **段锁+HDFS 流式 Bulk+瘦文档+限速**；详情先回 HDFS。Nginx 限流，合流在应用层。

---

## 12. 相关文档

- [02-interview-qa.md](./02-interview-qa.md) — 缓存击穿、限流、网关相关题；场景题 I4  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md) — 弹力与限流  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md) — Cache-Aside  


