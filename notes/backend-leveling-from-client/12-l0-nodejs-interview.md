# L0：Node.js 后端入门（面试向详解）

> 面向：**客户端转后端、主语言选 Node/TypeScript**。  
> 目标：面试能讲清「为什么用 Node、事件循环、异步、中间件、进程与连接池、和浏览器 JS 的差别」，并能落到最小 HTTP 服务。  
> 配合：[11-http-server-nodejs.md](./11-http-server-nodejs.md)、[03-minimal-backend-stack.md](./03-minimal-backend-stack.md)、日志场景 [10](./10-log-cold-hot-es-nginx.md) / 面试 I5。

---

## 1. 面试开场：为什么选 Node 做后端？

**高分答法（客户端背景）**：

> 我来自 RN/前端，用 Node/TS 迁移成本低，能快速把「会调 API」做成「会提供 API」。Node 适合 **IO 密集**（等 DB/Redis/ES/HTTP），和客户端异步网络模型同构。代价是要补：**事件循环边界、类型、多实例状态、连接池总账**，不能把浏览器思维原样搬过来。

| 适合 | 不太适合（入门阶段如实说） |
|------|---------------------------|
| API 网关后的 BFF、CRUD、日志查询、实时消息（配合 WS） | CPU 重（大包同步 JSON、视频转码）→ Worker / 别的语言 |
| 快速原型、TS 全栈 | 强一致复杂事务中枢也可做，但团队栈更常是 Java/Go |

---

## 2. 和浏览器 JS 的关键差别（必考）

| 点 | 浏览器 | Node 后端 |
|----|--------|-----------|
| 全局 | `window` / DOM | **无 DOM**；有 `process`、`Buffer`、`fs`、`net` |
| 模块 | 历史杂、现 ESM | CommonJS `require` / ESM `import`（面试说清项目用哪种） |
| 入口 | 页面生命周期 | **进程**：一个 `node app.js` 监听端口 |
| 保密 | 密钥不能放前端 | 密钥只在服务端环境变量 |
| 并发 | 多 Tab 多线程感 | **单线程 JS + 线程池做 IO**；CPU 重活会堵所有请求 |

口试金句：Node 不是「单线程什么都不能做」，而是 **JS 回调在单线程；libuv 用线程池/epoll 做 IO**。

---

## 3. 事件循环（Event Loop）——高频深挖

### 3.1 一句话

> 主线程跑完同步代码后，按阶段取任务队列执行；`fs`/网络完成时把回调丢进队列，**不会并行执行两段 JS**。

### 3.2 面试常画的简化图

```text
同步代码
  → process.nextTick / microtask（Promise.then）
  → 定时器 timers（setTimeout）
  → IO 回调（网络、文件）
  → setImmediate / close
  → 再转下一圈
```

### 3.3 会翻车的点（结合本项目）

| 行为 | 后果 | 改法 |
|------|------|------|
| 请求里同步 `JSON.parse` 巨大日志包 | 所有用户 P99 爆炸 | 流式按行解析 |
| 同步 `fs.readFileSync` | 卡死 loop | 只用 async API |
| 超大 `JSON.stringify` 响应 | 同样卡 | 分页、截断 preview |
| 在 handler 里算 cropto 重哈希 | 卡 | `crypto` 异步或 Worker |

**监控**：`event loop lag`（延迟多少毫秒才轮到下一圈）。

### 3.4 口述题：「Node 如何处理高并发？」

> 不是一人一线程，而是单线程非阻塞 IO：一万个连接大多在等，回调来了才占 CPU 一小段。并发上限受 **event loop、内存、下游连接池** 约束，所以要限流、`maxSockets`、避免重 CPU。

---

## 4. 异步模型（Promise / async-await / 错误）

### 4.1 后端标配

```js
async function listLogs(req, reply) {
  try {
    const rows = await es.search(...);
    return rows;
  } catch (e) {
    req.log.error(e);
    throw e; // 交给统一 error handler
  }
}
```

### 4.2 面试要点

| 题 | 答 |
|----|----|
| 回调地狱怎么解决 | Promise / async-await；仍要统一错误处理 |
| `await` 会阻塞进程吗 | **只阻塞当前异步函数的后续**，不阻塞其它请求的 JS（除非你写同步重活） |
| 未处理的 Promise rejection | 可能打日志或在严模式下影响进程 → 框架层 + `unhandledRejection` 观测 |
| 并行请求下游 | `Promise.all` / `allSettled`；注意下游连接池被放大 |

### 4.3 与 HTTP 超时

```text
客户端超时 < Nginx read < Node 对下游的 timeout
AbortSignal / 取消：客户端断开后别无限占着 ES 连接
```

---

## 5. 模块、项目结构、TypeScript

### 5.1 面试怎么说模块

- **CommonJS**：`require` / `module.exports`（老项目）  
- **ESM**：`import` / `export`（`"type":"module"`）  
- 循环依赖：初始化顺序问题 → 分层、延迟 require  

### 5.2 L0 推荐目录

```text
src/
  app.ts          # 创建服务器、挂路由
  config.ts       # 环境变量，缺必填即退出
  routes/
  middleware/     # auth、idempotency、requestId
  services/       # 业务，不碰 req/res
  clients/        # es.ts redis.ts 单例
```

### 5.3 TypeScript 面试加分

> 后端用 TS 是为了 **契约与重构安全**，不是为了「像前端一样写界面」。DTO、错误码、配置用类型固定；`any` 留给边界并尽快收窄。

---

## 6. HTTP 服务：Express vs Fastify（面试选型）

| | Express | Fastify |
|--|---------|---------|
| 生态 | 最大、教程多 | 更快、schema 校验一流 |
| 中间件 | `(req,res,next)` | hook / preHandler |
| L0 建议 | 都会一点 | 新项目可 Fastify |

**共同点（必讲）**：

1. Method + Path → Handler  
2. 中间件链 ≈ 客户端拦截器（鉴权 → 日志 → 业务）  
3. 统一错误与 JSON  
4. `listen(port)` + 优雅关闭（`SIGTERM` 停接新请求、等飞行中请求）

---

## 7. 中间件链（对照客户端拦截器）

```text
Request
  → requestId
  → accessLog
  → auth          // Authorization
  → idempotency   // POST
  → handler
  → errorHandler
Response
```

面试映射：

| 客户端 | Node |
|--------|------|
| OkHttp Interceptor | middleware / hook |
| 统一加 Token | auth 中间件 |
| 统一解包错误 | errorHandler + 状态码 |

详见 [11-http-server-nodejs.md](./11-http-server-nodejs.md)。

---

## 8. 配置、日志、探活（L0 最小栈用 Node 实现）

### 8.1 配置

```js
function required(name) {
  const v = process.env[name];
  if (!v) {
    console.error(`missing env ${name}`);
    process.exit(1);
  }
  return v;
}
export const config = {
  port: Number(process.env.PORT ?? 3000),
  esUrl: required('ES_URL'),
  redisUrl: required('REDIS_URL'),
};
```

面试：**十二要素**——配置进环境变量；密钥不进镜像层乱写。

### 8.2 结构化日志

一行 JSON：`time, level, request_id, method, path, status, latency_ms, err`。  
框架：`pino`（Fastify 默认）优于 `console.log` 字符串拼接。

### 8.3 健康检查

```js
app.get('/healthz', async () => ({ ok: true }));
// L1+：readiness 再查 Redis/ES ping
```

---

## 9. 进程、cluster、连接池（面试常连问）

### 9.1 默认单进程

```text
总 ES 连接 ≈ 1 × maxSockets
```

### 9.2 多进程

- `cluster` / PM2 `instances`  
- K8s：多 Pod，每 Pod 常 **1 进程**  

```text
总连接 ≈ 进程数(或 Pod 数) × maxSockets
```

### 9.3 单例 Client

```js
// clients/es.ts —— 模块级单例，禁止在 handler 里 new Client()
export const es = new Client({ node, agent: { https: new Agent({ maxSockets: 32 }) } });
```

### 9.4 多实例不能共享的内存

| 能放进程内存 | 必须外置 |
|--------------|----------|
| 单次请求内状态 | Session、幂等结果、warm、分布式锁 |
| singleflight Map（仅本进程） | 跨 Pod 用 Redis SET NX |

---

## 10. L0 用 Node 做项目 A 的验收话术

面试官：你 L0 项目怎么证明会做服务端？

> 我用 Node+TS 做了书签/待办 API：JWT 鉴权、CRUD、cursor 分页、请求 ID 日志、`/healthz`、Docker Compose。故意用客户端弱网重试 POST，靠 Idempotency-Key 保证不双写；GET 可重试。能讲清 4xx/5xx 和超时预算。

---

## 11. 高频面试题速答（L0 Node）

### Q1. Node 是单线程的，怎么撑住线上流量？

非阻塞 IO + 多连接等待；水平扩展多进程/多 Pod；下游连接池与限流；CPU 密集卸载。

### Q2. `require` 和 `import` 区别？

加载时机、循环依赖、是否严格 ESM；项目统一一种。

### Q3. 如何优雅重启？

`SIGTERM` → 停止 listen → 等 in-flight（设上限）→ 关闭 DB/Redis → exit。K8s 配 `preStop`/grace period。

### Q4. 内存泄漏常见原因？

全局缓存无上限、未结束的监听器、闭包抓住大 buffer、singleflight Map 未 delete。

### Q5. 和 Java Spring 比，Node 弱在哪？

你主动说：CPU、强类型生态、部分企业中间件；并说你如何用 TS、测试、规范补。

### Q6. 日志查询场景为何 Node 合适？

IO 等待 HDFS/ES/Redis；用 singleflight、流式解析、Bulk 异步；注意 loop 与连接池（见 I5）。

### Q7. 什么是中间件？

请求管道上可插拔的函数，处理横切逻辑（鉴权、日志、幂等），与业务 handler 分离。

### Q8. POST 为何要 Idempotency-Key？

HTTP 语义上 POST 不幂等；客户端超时重试会导致双份副作用；服务端用键去重。

### Q9. 如何限制对 ES 的连接？

全局 Client + `Agent.maxSockets` + 可选 p-limit；总账=进程数×maxSockets。

### Q10. `process.nextTick` 和 `setImmediate`？

nextTick 更高优先（耗尽风险）；setImmediate 偏下一轮 check 阶段。业务代码优先 async/await，少炫技。

---

## 12. 学习路径（2～4 周 L0）

| 周 | 做什么 | 面试能秀什么 |
|----|--------|--------------|
| 1 | HTTP 方法/状态码 + 最小 Fastify/Express | 讲清安全/幂等 |
| 2 | JWT、错误模型、结构化日志、Docker | 项目 A 可演示 |
| 3 | Idempotency-Key、限流 429、超时预算 | 弱网重试故事 |
| 4 | 连接池、健康检查、读写分离概念预习 | 为 L1 SQL 铺路 |

---

## 13. 收口（面试 30 秒版）

> Node 适合 IO 型 API；核心是事件循环别被同步重活堵住。用 TS+框架做 Method 路由与中间件，配置走环境变量，Client 单例控 `maxSockets`。POST 配幂等键，错误用对 4xx/5xx。多实例状态放 Redis。客户端经验帮我理解重试与超时，服务端要把它变成契约。

---

## 14. 相关文档

- [11-http-server-nodejs.md](./11-http-server-nodejs.md)  
- [03-minimal-backend-stack.md](./03-minimal-backend-stack.md)  
- [02-interview-qa.md](./02-interview-qa.md) I5  
- [10-log-cold-hot-es-nginx.md](./10-log-cold-hot-es-nginx.md)  
