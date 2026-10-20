# HTTP 再学一遍（服务端视角）+ Node.js 落地

> 对应练级 L0「HTTP 再学一遍」。目标：从「我会用 OkHttp/URLSession 调接口」变成「我设计的接口让客户端敢重试、状态码说得清、Header 有契约」。  
> 实现语言以 **Node.js / TypeScript** 为主（Express / Fastify 示意通用）。

---

## 0. 先分清三个词

| 词 | 含义 | 例子 |
|----|------|------|
| **安全（Safe）** | 不改变服务器资源状态（只读） | GET、HEAD |
| **幂等（Idempotent）** | 同一请求做 1 次和 N 次，**副作用相同** | GET、PUT、DELETE；带幂等键的 POST |
| **可缓存** | 响应可被缓存复用 | 常配合 GET + 缓存头 / ETag |

客户端弱网会 **重试**。服务端若方法语义乱、状态码乱、无幂等，重试就会制造双单、双扣款。

---

## 1. 方法语义（服务端怎么选）

### 1.1 对照表

| 方法 | 安全 | 幂等 | 典型用途 | 服务端注意 |
|------|------|------|----------|------------|
| **GET** | ✅ | ✅ | 读列表/详情 | 禁止在 GET 里改库、发短信、扣库存 |
| **HEAD** | ✅ | ✅ | 只要头、不要 body | 与 GET 同路径语义一致 |
| **PUT** | ❌ | ✅ | 用客户端给的完整表示 **覆盖** 资源 | 同一 URL 多次 PUT = 同一最终状态 |
| **DELETE** | ❌ | ✅ | 删除资源 | 删一次与删多次：资源都「不在了」（可 200/204 或二次 404，团队统一） |
| **POST** | ❌ | ❌（默认） | 创建、触发动作、非幂等写 | **默认可重试不安全** → 要 `Idempotency-Key` / 业务唯一键 |
| **PATCH** | ❌ | 看实现 | 部分更新 | 规范上不保证幂等；做成幂等需约定 |

### 1.2 设计口诀

```text
只读          → GET
创建新资源      → POST /items          （配幂等键）
按 id 整份覆盖 → PUT  /items/{id}
按 id 删除    → DELETE /items/{id}
触发「做一次」 → POST /orders/{id}/pay （更要幂等键）
```

### 1.3 和客户端重试的关系

| 客户端行为 | 服务端必须 |
|------------|------------|
| GET 失败自动重试 | GET 真只读；5xx/网络错可重试；4xx 一般别盲目重试 |
| POST 下单超时重试 | **幂等键或订单唯一约束**，否则双单 |
| 收到 502/503 | 可退避重试；你返回要对：过载用 503 + `Retry-After` |
| 收到 400/401/403/404/409 | **不要当网络抖**去重试同一坏请求 |

---

## 2. 状态码分层

### 2.1 三层怎么用

| 层 | 含义 | 调用方怎么做 |
|----|------|--------------|
| **2xx** | 成功 | 解析 body，更新 UI |
| **4xx** | **客户端错**（请求本身有问题） | 改请求 / 重新登录；**默认不重试** |
| **5xx** | **服务端错**（我这边挂了或过载） | 可退避重试；看 `Retry-After` |

常见码（服务端视角）：

| 码 | 何时用 | 备注 |
|----|--------|------|
| 200 | 成功带 body | GET/POST 结果 |
| 201 | 创建成功 | POST 创建；可带 `Location` |
| 204 | 成功无 body | DELETE/PUT 常用 |
| 304 | 未修改 | 配合 ETag 条件 GET |
| 400 | 参数/JSON 不合法 | 校验失败 |
| 401 | 未认证 | 缺/坏 Token |
| 403 | 已认证无权限 | 别用 401 糊弄 |
| 404 | 资源不存在 | |
| 409 | 冲突 | 唯一键冲突、状态机不允许 |
| 422 | 语义对但业务校验失败 | 看团队是否与 400 合并 |
| 429 | 限流 | **必带** `Retry-After` 更佳 |
| 500 | 未处理异常 | 少用；应观测告警 |
| 502/503/504 | 网关/过载/上游超时 | Nginx/Node 超时预算要对齐 |

### 2.2 统一错误 JSON（建议）

```json
{
  "error": {
    "code": "IDEMPOTENCY_CONFLICT",
    "message": "Idempotency-Key reused with different body",
    "request_id": "01JB…"
  }
}
```

规则：所有 4xx/5xx 尽量同结构；`request_id` 与日志一致，方便客户端甩锅给你查。

---

## 3. 关键 Header（服务端要认、要回）

### 3.1 `Authorization`

```text
Authorization: Bearer <JWT>
```

服务端：校验签名/过期 → 注入 `userId` → 再进业务。  
失败：401（未带/坏票）vs 403（票对但没权）。

### 3.2 `Idempotency-Key`（POST 救星）

客户端为「这一次用户动作」生成唯一键（UUID），重试带 **同一把键**：

```text
POST /api/orders
Idempotency-Key: 7c9e…-…
```

服务端：

```text
若 key 未见过 → 执行业务 → 存 (key → response/status)
若 key 见过且请求体相同 → 直接返回上次响应（真幂等）
若 key 见过但请求体不同 → 409
```

存储：Redis（TTL 24h）或 DB 唯一表。

### 3.3 `ETag` / `If-None-Match`（缓存与防丢更新）

```text
响应：ETag: "v12"
再请求：If-None-Match: "v12"  → 未变则 304
写更新：If-Match: "v12"     → 版本不对则 412（防丢更新）
```

列表日志场景也可对「整页结果」短缓存，但权限 key 必须带 `userId`。

### 3.4 分页头 / 限流头

| Header | 方向 | 含义 |
|--------|------|------|
| `Link` 或 body 内 `nextCursor` | 响应 | 下一页（本项目日志列表更常用 cursor） |
| `X-Request-Id` / `X-Request-ID` | 请求或响应 | 追踪；无则服务端生成 |
| `Retry-After` | 响应 | 429/503 时建议多久后重试（秒） |
| `X-RateLimit-Limit/Remaining` | 响应 | 可选，告知配额 |

Nginx `$request_id` 可透传成 `X-Request-Id`，Node 写入日志。

---

## 4. 客户端经验 → 服务端契约

你在端上写过的东西，服务端要这样接：

| 客户端做法 | 服务端配合 |
|------------|------------|
| 超时重试 GET | GET 只读；超时返回 504/放弃，别半写 |
| 超时重试 POST | 必须幂等键或唯一约束 |
| Token 过期刷新 | 过期 JWT → 401；刷新接口清晰 |
| 429 退避 | 认真返回 429 + Retry-After |
| 幂等重试同一单 | 同 Idempotency-Key 返回同一结果，不要第二次再扣款 |
| 取消请求 | 服务端可用 AbortSignal；回填类可与响应脱钩异步做 |

---

## 5. Node.js 详细落地

### 5.1 路由按 Method 分开（Fastify 示意）

```js
// GET 只读
fastify.get('/api/logs', { preHandler: [auth] }, listLogs);

// POST 创建：强制幂等键
fastify.post('/api/orders', { preHandler: [auth, requireIdempotencyKey] }, createOrder);

// PUT 覆盖
fastify.put('/api/items/:id', { preHandler: [auth] }, putItem);

// DELETE
fastify.delete('/api/items/:id', { preHandler: [auth] }, deleteItem);
```

Express 同理：`app.get` / `app.post`，**不要**一个 handler 里用 query `?action=delete` 冒充 REST。

### 5.2 统一响应与错误中间件

```js
class HttpError extends Error {
  constructor(status, code, message) {
    super(message);
    this.status = status;
    this.code = code;
  }
}

fastify.setErrorHandler((err, req, reply) => {
  const status = err.status ?? 500;
  const request_id = req.id;
  req.log.error({ err, request_id }, 'request failed');
  reply.code(status).send({
    error: {
      code: err.code ?? (status >= 500 ? 'INTERNAL' : 'BAD_REQUEST'),
      message: status >= 500 ? 'internal error' : err.message,
      request_id,
    },
  });
});
```

业务里：`throw new HttpError(409, 'CONFLICT', 'duplicate order')`。

### 5.3 Authorization 中间件

```js
async function auth(req) {
  const h = req.headers.authorization;
  if (!h?.startsWith('Bearer ')) {
    throw new HttpError(401, 'UNAUTHORIZED', 'missing bearer token');
  }
  try {
    req.user = await verifyJwt(h.slice(7));
  } catch {
    throw new HttpError(401, 'UNAUTHORIZED', 'invalid token');
  }
}
```

### 5.4 Idempotency-Key 中间件（POST）

```js
import crypto from 'node:crypto';

function hashBody(body) {
  return crypto.createHash('sha256').update(JSON.stringify(body)).digest('hex');
}

async function requireIdempotencyKey(req) {
  const key = req.headers['idempotency-key'];
  if (!key || typeof key !== 'string') {
    throw new HttpError(400, 'IDEMPOTENCY_KEY_REQUIRED', 'Idempotency-Key required');
  }
  req.idempotencyKey = key;
}

async function withIdempotency(req, reply, handler) {
  const key = `idem:${req.user.sub}:${req.idempotencyKey}`;
  const bodyHash = hashBody(req.body);
  const cached = await redis.get(key);
  if (cached) {
    const prev = JSON.parse(cached);
    if (prev.bodyHash !== bodyHash) {
      throw new HttpError(409, 'IDEMPOTENCY_CONFLICT', 'key reused with different body');
    }
    return reply.code(prev.status).send(prev.body);
  }

  const result = await handler(); // { status, body }
  await redis.setex(key, 86400, JSON.stringify({
    bodyHash,
    status: result.status,
    body: result.body,
  }));
  return reply.code(result.status).send(result.body);
}

// 使用
fastify.post('/api/orders', { preHandler: [auth, requireIdempotencyKey] }, async (req, reply) => {
  return withIdempotency(req, reply, async () => {
    const order = await createOrderTx(req.user.sub, req.body);
    return { status: 201, body: order };
  });
});
```

与 DB 唯一约束可双保险：`UNIQUE(user_id, idempotency_key)`。

### 5.5 ETag 条件 GET（示意）

```js
fastify.get('/api/items/:id', async (req, reply) => {
  const item = await db.item.find(req.params.id);
  if (!item) throw new HttpError(404, 'NOT_FOUND', 'item not found');

  const etag = `"${item.version}"`;
  if (req.headers['if-none-match'] === etag) {
    return reply.code(304).send();
  }
  reply.header('ETag', etag);
  return item;
});
```

### 5.6 限流响应

```js
// 应用层令牌桶触发时
reply
  .header('Retry-After', '2')
  .code(429)
  .send({ error: { code: 'RATE_LIMITED', message: 'too many requests', request_id: req.id } });
```

Nginx `limit_req` 也可直接 429；两边策略要一致，避免一端松一端死。

### 5.7 分页（cursor）响应形状

```js
return {
  items,
  nextCursor: next ?? null,  // 不要暴露裸 SQL offset
};
// 或头：Link: <...&cursor=xxx>; rel="next"
```

改筛选条件必须让客户端丢弃旧 cursor（服务端校验 `filtersHash`）。

### 5.8 和日志查询项目的对应

| HTTP 点 | 本项目 |
|---------|--------|
| GET `/api/logs` | 只读列表；可短缓存；可重试 |
| 查询无 body 副作用 | 禁止在 GET 里触发「无锁狂写」；回填要 singleflight |
| 4xx | 缺 userId/时间窗 → 400；无权限 → 403 |
| 5xx / 超时 | HDFS/ES 挂 → 503/504；客户端可退避 |
| 限流 | Nginx + 应用 429 |
| 幂等 | 列表 GET 天然幂等；回填写 ES 靠 `_id` 幂等（不是 HTTP 幂等键，但是同一思想） |

### 5.9 Node 易错清单

| 错 | 后果 | 改 |
|----|------|-----|
| GET handler 里改库 | 重试变双写 | 写操作换 POST/PUT |
| POST 无幂等键 | 弱网双单 | Idempotency-Key + 存储 |
| 啥错都 500 | 客户端乱重试 | 分 4xx/5xx |
| 校验失败仍 200 + `{ok:false}` | 客户端难写 | 用真状态码 |
| 每请求 `new Client()` | 连接池炸 | 单例 + maxSockets |
| 忽略 `req.aborted` / AbortSignal | 客户端已走你还算 | 长 IO 可取消或与回填脱钩 |

---

## 6. 最小自测清单

- [ ] GET 连续打 10 次无多余写库  
- [ ] POST 带同一 Idempotency-Key 打 10 次只有 1 次业务副作用  
- [ ] 换 body 同 Key → 409  
- [ ] 坏 Token → 401；无权限 → 403  
- [ ] 限流 → 429 且有 Retry-After（或文档说明）  
- [ ] 错误 JSON 含 `request_id`，能对上日志  

---

## 7. 面试收口

> HTTP 方法告诉客户端「能不能放心重试」；状态码告诉「怪谁、能不能重试」；Idempotency-Key / ETag / 限流头是契约。  
> Node 侧用 Method 路由 + 鉴权中间件 + 幂等中间件 + 统一错误层落地；POST 默认不幂等，必须显式设计。

---

## 8. 相关文档

- [01-leveling-guide.md](./01-leveling-guide.md) — L0  
- [03-minimal-backend-stack.md](./03-minimal-backend-stack.md) — 路由/日志/探活  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md) — 重试与幂等  
- [02-interview-qa.md](./02-interview-qa.md) — 相关面试题  
