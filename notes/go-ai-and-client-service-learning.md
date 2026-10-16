# 基于 Go 做 AI 调用服务 + 客户端服务：知识体系与学习指南

> 目标读者：要从 0 搭「面向 App 的后端」和「AI 调用中台/服务」的工程师。  
> 产出形态：两套可协作的 Go 服务——**Client API（客户端服务）** 与 **AI Gateway / AI Worker（AI 调用服务）**。  
> 关联：社交 App 移动端分层见同仓库架构笔记（若已合并）。  
> 交互版学习路径：[Go AI与客户端服务学习路径](https://cursor.com/canvas/store-3f310d1e-80b7-486b-8c4a-9718b9ab9225/93ac882e-733e-4a30-82a5-ad100a4810c3)。

---

## 0. 先建立全局图

```
┌──────────── Mobile / Web ────────────┐
│  AppShell · Feed · IM · AI 对话 UI    │
└───────────────┬──────────────────────┘
                │ HTTPS / WSS
                ▼
┌──────────── Client API（Go）─────────┐
│  鉴权 · 业务 API · BFF · 实时网关     │
│  Feed / 关系 / IM / 媒资 / AI 门面    │
└───────┬──────────────────┬───────────┘
        │                  │ 内部 RPC / MQ
        ▼                  ▼
┌──────────────┐   ┌───────────────────┐
│ 领域业务服务  │   │ AI 调用服务（Go）   │
│ 用户/内容/IM │   │ Provider 适配       │
└──────────────┘   │ 流式 · 限流 · 审计  │
                   │ Prompt · Tool · RAG │
                   └─────────┬─────────┘
                             │
                   OpenAI / 国产大模型 / 自建推理
```

**两个服务为什么拆开？**

| | Client API | AI 调用服务 |
|---|---|---|
| 调用方 | App / Web | 主要是内部服务（也可受控对外） |
| 稳定性诉求 | 低延迟、高 QPS、强一致业务 | 长耗时、流式、供应商抖动 |
| 扩缩容 | 随 DAU 水平扩展 | 随 token/并发独立扩展 |
| 密钥 | 用户态 JWT/Session | 模型 API Key、企业网关凭证 |
| 失败形态 | 4xx/业务错误 | 超时、限流、内容安全、模型降级 |

学习时按「先 Client 骨架 → 再 AI 调用专用能力 → 再拼到业务」推进，而不是一上来就堆 Agent 框架。

---

## 1. 学习目标（学完应能做什么）

1. 用 Go 搭一套可上线的 **HTTP/gRPC 客户端 API**：鉴权、分页、错误码、观测、配置。  
2. 用 Go 实现 **AI 调用服务**：多模型适配、SSE/流式、重试熔断、用量与审计、Prompt 版本。  
3. 能把 AI 能力接到客户端链路：`App → Client API → AI 服务 → 模型 → 流式回传`。  
4. 懂边界：哪些逻辑在客户端 Domain、哪些在 Client API、哪些必须进 AI 服务。

---

## 2. 知识地图（总览）

```
A. Go 语言与工程基础
B. 网络与 API 协议（HTTP/JSON、SSE、WebSocket、gRPC）
C. 客户端服务（业务 API / BFF / 实时）
D. AI 调用服务（Provider、流式、编排、安全、成本）
E. 数据与中间件（PG/MySQL、Redis、MQ、对象存储）
F. 安全、合规、内容安全
G. 可观测性与 SRE
H. 部署与交付（容器、K8s、CI/CD）
I. 与移动端契约协作
```

下文按块展开：每块含「要学什么 / 为什么 / 做到什么程度 / 实践任务」。

---

## 3. A. Go 语言与工程基础

### 3.1 语言核心（必须扎实）

| 主题 | 要掌握的点 | 做到什么程度 |
|---|---|---|
| 类型与接口 | struct、interface、类型断言、泛型基础 | 能设计 `Provider` / `Repository` 接口 |
| 并发 | goroutine、channel、`select`、`errgroup`、`sync`、context 取消 | 能安全做并发扇出与超时取消 |
| `context.Context` | 超时、取消、值传递边界（勿乱塞业务） | 全链路传 ctx；下游可取消 |
| 错误处理 | `error` 包装、`errors.Is/As`、哨兵错误 vs 业务错误 | API 层能映射稳定错误码 |
| 模块 | `go.mod`、语义化版本、replace、私有 module | 多服务复用 `pkg/` 或独立 module |
| 测试 | table-driven、`httptest`、fake/mock、race detector | 核心路径有单测；关键并发加 `-race` |

**建议练习**

- 写一个带超时的并行 `fan-in` 聚合器（模拟多路召回）。  
- 实现可取消的 worker pool（为后续 AI 任务队列打底）。

### 3.2 工程结构（建议约定）

```
cmd/
  client-api/main.go
  ai-gateway/main.go
internal/
  config/
  transport/http|grpc
  service/          # 应用服务 / usecase
  domain/           # 领域模型（可选）
  adapter/          # DB、Redis、LLM Provider
  middleware/
pkg/                # 可对外复用的库（错误码、鉴权辅助）
api/                # OpenAPI / proto
deploy/
```

要学：**整洁架构在 Go 里的落地**——`transport → service → adapter`，依赖向内；不要在 handler 里直接调 OpenAI SDK 散落各处。

### 3.3 常用标准库 / 生态

| 类别 | 推荐了解 |
|---|---|
| HTTP | `net/http`（必懂）、路由可用 `chi` / `echo` / `gin` / `fiber`（选一吃透） |
| 配置 | `env` + 文件；了解 `viper` 但勿过度 |
| 日志 | `slog`（Go 1.21+）或 zap |
| 校验 | `go-playground/validator` 或手动校验 |
| 迁移 | `golang-migrate` / goose |
| ORM/SQL | 先 `database/sql` + sqlc；再用 GORM（知利弊） |

---

## 4. B. 网络与 API 协议

### 4.1 面向客户端的 HTTP API

必须掌握：

- RESTful 资源设计 vs RPC 风格（团队统一即可）  
- **分页**：cursor 分页（社交 Feed 强烈推荐）  
- **幂等**：`Idempotency-Key` / `client_request_id`  
- **错误体**：稳定 `code` + `message` + `request_id`  
- **版本**：`/v1`；破坏性变更策略  
- **鉴权**：Bearer JWT / Session；刷新令牌旋转  
- **限流**：用户级 / IP 级 / 设备级  

**OpenAPI 3**：当作与 iOS/Android 的契约源，而不是事后补文档。

### 4.2 流式：SSE 与分块传输

AI 回复几乎必用流式。Go 侧要学：

- `http.Flusher`、禁用缓冲中间件  
- SSE 事件格式：`event:` / `data:` / 心跳 comment  
- 客户端断线检测：`ctx.Done()`、写失败退出  
- 与移动端：OkHttp/URLSession 读流；注意后台保活限制  

对比：

| 方式 | 适用 |
|---|---|
| SSE（HTTP 单向流） | AI 文本流式输出（优先） |
| WebSocket | IM、双向实时 |
| gRPC stream | 服务间内部流式 |

### 4.3 WebSocket（客户端服务常见）

- 握手、心跳、重连、房间/连接注册表  
- 水平扩展：连接粘滞 vs Redis Pub/Sub 扇出  
- 与业务消息序（`seq` / `client_msg_id`）配合——协议在业务层定，传输层只保证送达帧  

### 4.4 gRPC（服务间强烈建议）

- proto3、代码生成、错误详情、deadline  
- Client API → AI 服务 用 gRPC；对外仍 HTTP/SSE  
- 要学：拦截器（鉴权、日志、metric）、负载均衡、健康检查  

---

## 5. C. 客户端服务（Client API）知识详解

「客户端服务」= App 直连的后端门面，常兼 BFF。社交场景下建议按域拆，学习时先做**模块化单体**，再拆微服务。

### 5.1 业务域能力清单（要会设计表 + API）

| 域 | 核心知识 | 典型接口 |
|---|---|---|
| 账号 | 注册登录、JWT、刷新、风控钩子、踢下线 | `POST /v1/auth/login` |
| 关系 | 关注边、粉丝列表、计数一致性 | `POST /v1/relations/follow` |
| Feed | 写扩散/读扩散基础、cursor、互动计数 | `GET /v1/feed/timeline` |
| 内容 | 帖子模型、可见性、软删 | `POST /v1/posts` |
| 媒资 | 预签名上传、回调、转码状态 | `POST /v1/media/upload-token` |
| IM | 会话、消息、seq、补洞、未读 | WS + `GET /v1/im/messages` |
| 推送 | device token、厂商通道抽象 | 内部投递 |
| AI 门面 | 会话、配额、把请求转给 AI 服务 | `POST /v1/ai/chats/{id}/messages` |

### 5.2 鉴权与会话（必学深）

- Access / Refresh 分离；Refresh 旋转与重放检测  
- 中间件：解析 JWT → 注入 `user_id`  
- 服务间：mTLS 或内部 service token，**不要**把用户 JWT 随便透传外网模型  
- 权限：资源归属校验（只能删自己的帖）  

### 5.3 数据层知识

| 主题 | 学什么 |
|---|---|
| 关系型建模 | 用户、关系边、帖子、评论、会话、消息表 |
| 索引 | 时间线、会话消息 `(session_id, seq)` 唯一索引 |
| 事务 | 发帖 + 计数；关注 + 双边计数（或异步最终一致） |
| 缓存 | Redis：session、热点帖、未读；缓存穿透/击穿/雪崩 |
| 对象存储 | S3/OSS 预签名；回调校验 |
| 搜索（后置） | OpenSearch/ES 基础 |

### 5.4 实时与推送

- 连接网关与业务服务分离的思想  
- 在线状态、未读聚合  
- 推送：与 APNs/FCM 的边界；Go 侧任务投递与失败重试  

### 5.5 BFF 职责边界

Client API **应该**：

- 聚合移动端需要的字段（减少 round-trip）  
- 统一错误码与分页形态  
- 做用户级鉴权、配额入口  

Client API **不应该**：

- 直接持有模型厂商 API Key 并散落调用（应走 AI 服务）  
- 实现复杂 Prompt 编排与工具循环（放 AI 服务）  
- 把移动端 Domain 同步逻辑搬到服务端重复一套「假客户端」  

### 5.6 实践任务（Client API）

**里程碑 C1**：用户注册登录 + 发帖 + Feed cursor 列表 + 统一错误码与 `request_id`。  
**里程碑 C2**：图片预签名上传 + 发帖引用 media_id。  
**里程碑 C3**：IM 单聊（WS + 历史补洞 + 未读）。  
**里程碑 C4**：`/v1/ai/...` 门面，把流式响应透传给 App。

---

## 6. D. AI 调用服务知识详解（重点）

这是「基于 Go 做 AI 调用」的核心课表。

### 6.1 服务定位

AI 调用服务 = **模型访问的统一出口**，对内提供：

1. Chat / Completion（同步 + 流式）  
2. 多 Provider 适配与路由  
3. 限流、超时、重试、降级、熔断  
4. Prompt 模板与版本  
5. Tool / Function Calling 编排（可选 Agent）  
6. 用量计量、审计日志、内容安全钩子  
7. （可选）RAG：检索 + 组装上下文  

### 6.2 Provider 适配层（必学）

设计稳定内部协议，隔离厂商差异：

```go
type ChatClient interface {
    Chat(ctx context.Context, req ChatRequest) (*ChatResponse, error)
    ChatStream(ctx context.Context, req ChatRequest) (<-chan StreamEvent, error)
}

type ChatRequest struct {
    Model       string
    Messages    []Message
    Temperature *float32
    MaxTokens   *int
    Tools       []ToolSpec
    User        string // 用于供应商侧滥用追踪，勿传隐私明文
    Metadata    map[string]string
}
```

要学的知识点：

| 主题 | 细节 |
|---|---|
| OpenAI 兼容协议 | Chat Completions / Responses；很多国产模型提供兼容层 |
| 鉴权 | API Key、Azure AD、自定义网关 |
| 模型路由 | 按场景（便宜/强/长上下文/国内合规）选模型 |
| 超时 | 连接超时 vs 首 token 超时 vs 总超时（流式要分开） |
| 重试 | 仅对 429/5xx/网络错误；尊重 `Retry-After`；指数退避 + 抖动 |
| 幂等 | 业务侧 `request_id`；避免「超时后重试导致重复扣费/重复副作用」 |

### 6.3 流式调用（必学深）

完整链路：

```
模型 SSE → AI 服务解析 delta → 内部事件 →（gRPC stream 或 HTTP SSE）→ Client API → App
```

要点：

- 解析 vendor SSE，归一成自己的 `StreamEvent{Type, Delta, Usage, Error}`  
- **背压**：App 读得慢时如何处理（缓冲上限、断开上游）  
- **取消传播**：用户点停止 → Client 取消 → AI 服务 cancel ctx → 停上游  
- **心跳**：防代理断空闲连接  
- **usage 落账**：有的供应商在最后一帧给 usage，要可靠收集  

### 6.4 Prompt 工程在服务里的形态

不要只把 Prompt 当字符串塞代码：

| 知识 | 实践 |
|---|---|
| 系统/开发者/用户角色 | 模板分层，防用户覆写系统指令 |
| 模板引擎 | 文本模板 + 变量校验；禁止未转义拼接导致注入 |
| 版本管理 | `prompt_id + version`；灰度与回滚 |
| 输出约束 | JSON schema / 工具调用，减少胡言 |
| 防注入 | 分隔符、权限内工具、不可执行任意指令 |

### 6.5 Tool Calling / 轻量 Agent（进阶）

学习路径：

1. 单轮 tool：模型返回 `tool_calls` → Go 执行 → 回填 `tool` 消息 → 再请求  
2. 多轮循环：最大步数、超时、费用熔断  
3. 工具白名单与参数 schema 校验  
4. 副作用工具（下单、发消息）必须人工确认或强鉴权  

Go 里重点：**状态机清晰**，不要无界 goroutine 乱飞。

### 6.6 RAG 基础（按需）

| 组件 | 学什么 |
|---|---|
| 文档切分 | chunk 大小、重叠 |
| Embedding | 批量向量化、维度 |
| 向量库 | pgvector / Milvus / Qdrant 之一 |
| 检索 | topK、相似度、元数据过滤 |
| 组装 | 引用片段进上下文；控制 token 预算 |
| 评估 | 命中率、幻觉抽检 |

0→1 社交 App 的「AI 对话」若无知识库，可先不做 RAG。

### 6.7 限流、配额与成本

| 维度 | 策略 |
|---|---|
| 用户配额 | 日/月 token 或次数 |
| 并发 | 每用户/全局 in-flight 限制 |
| 供应商 RPM/TPM | 令牌桶；多 Key 池 |
| 成本 | 按 model 单价估算；预算告警 |
| 降级 | 主模型失败切便宜模型或返回友好错误 |

Go 实现线索：内存令牌桶 + Redis 分布式计数；Lua 脚本做原子扣减。

### 6.8 安全与合规（AI 特有）

- API Key 只在 AI 服务，经密钥管理（KMS/环境注入），不进日志  
- 输入输出过内容安全（厂商审核 API 或自建）  
- 审计：谁、何时、哪个模型、多少 token、是否命中安全策略（内容本身按合规存储/脱敏）  
- 提示词与用户隐私：最小化进模型；日志脱敏  
- 防 SSRF：工具若能发起 HTTP，必须 URL 白名单  

### 6.9 可观测（AI 特有指标）

- `ttft`（Time To First Token）  
- 流式总时长、token 入/出、错误分类（429/超时/内容拦截）  
- 按 model / prompt_version / scene 维度拆开  
- Trace：Client API span → AI 服务 span → 上游 HTTP  

### 6.10 实践任务（AI 服务）

**里程碑 A1**：单 Provider 同步 Chat + 统一错误 + 用量日志。  
**里程碑 A2**：SSE 流式 + 取消传播 + 心跳。  
**里程碑 A3**：第二 Provider 适配 + 模型路由 + 429 退避。  
**里程碑 A4**：Prompt 版本 + 用户配额（Redis）。  
**里程碑 A5**：Tool calling 最小环（如查天气假接口）+ 最大步数。  
**里程碑 A6**（可选）：RAG 最小闭环。

---

## 7. E. 数据与中间件（两边共用）

| 组件 | Client API 用法 | AI 服务用法 |
|---|---|---|
| PostgreSQL/MySQL | 用户/内容/IM | 对话会话、消息、用量、Prompt 版本 |
| Redis | 缓存、会话、未读、限流 | 配额、分布式锁、幂等键 |
| MQ（Kafka/NATS/Rabbit） | 异步粉丝通知、推送 | 异步总结、离线嵌入、重试队列 |
| 对象存储 | 图片视频 | 长文本/文件对话附件 |
| 向量库 | — | RAG |

要学：连接池、超时、重试、迁移、备份意识；**不要**在请求线程里做重 CPU embedding（丢 worker）。

---

## 8. F. 安全知识清单

| 层级 | 知识点 |
|---|---|
| 传输 | TLS、HSTS、证书 |
| 认证授权 | JWT、RBAC/资源鉴权、服务账号 |
| 输入 | 大小限制、JSON 深度、文件类型 |
| 密钥 | 环境变量/密钥管理；轮换 |
| 依赖 | `govulncheck`、供应链 |
| 隐私 | 日志脱敏、保留周期、删除权 |
| AI | 越狱/注入、工具 SSRF、内容安全 |

---

## 9. G. 可观测性与 SRE

统一学一套，两套服务共用规范：

1. **日志**：结构化 `slog`；强制 `request_id` / `trace_id` / `user_id(hashed)`  
2. **指标**：Prometheus（QPS、延迟直方图、错误率、AI ttft）  
3. **追踪**：OpenTelemetry  
4. **健康检查**：`/healthz` `/readyz`（依赖 DB/Redis 就绪）  
5. **配置热更**：功能开关；AI 模型路由可动态改  
6. **容量**：压测（k6/vegeta）；流式连接数与内存模型  

---

## 10. H. 部署与交付

| 主题 | 最低要求 |
|---|---|
| 容器 | 多阶段 Docker 构建；非 root；精简镜像 |
| 编排 | K8s Deployment + Service；就绪探针 |
| 发布 | 滚动更新；AI 服务注意长连接/流式 drain |
| CI | lint（golangci-lint）、test、vuln、镜像扫描 |
| 环境 | dev/staging/prod 配置隔离；密钥不进镜像 |
| 网关 | Ingress/API Gateway；超时对 AI 要单独加长 |

流式注意：网关/代理的 idle timeout、缓冲、HTTP/2 支持。

---

## 11. I. 与移动端协作知识

服务端同学也要懂这些客户端契约（与五层架构对齐）：

| 契约 | 说明 |
|---|---|
| OpenAPI | 字段、错误码、分页 |
| 深链 | AI 会话页路由 |
| 流式协议 | SSE 事件字段与结束帧 |
| 幂等 | `client_msg_id`（IM）与 AI `request_id` |
| 埋点 | `ai_ttft`、`ai_send_result`、发布成功率等 |
| 错误码表 | `AUTH_EXPIRED` / `AI_RATE_LIMITED` / `AI_CONTENT_BLOCKED`… |

移动端 L3 Domain（如 Conversation）消费的是**稳定事件**；不要让 App 直连模型厂商。

---

## 12. 两套服务的推荐内部 API

### 12.1 Client API → AI 服务（gRPC 示例语义）

```text
rpc CreateConversation(CreateConversationReq) returns (Conversation)
rpc SendMessage(SendMessageReq) returns (stream StreamEvent)
rpc GetUsage(GetUsageReq) returns (Usage)
```

`SendMessageReq` 含：`user_id`、`conversation_id`、`client_request_id`、`scene`、`messages` 或最新一条、`prompt_key`。

### 12.2 对 App 的 HTTP 门面

```text
POST /v1/ai/conversations
POST /v1/ai/conversations/{id}/messages:stream   # SSE
GET  /v1/ai/conversations/{id}
GET  /v1/ai/quota
```

Client API 负责：鉴权、配额预检、会话归属、审计用户动作；  
AI 服务负责：模型调用、流式、供应商错误归一、token 计量。

---

## 13. 分阶段学习路径（建议顺序）

### Phase 0 — Go 后端基本功（1 段）

- [ ] Go 语法 + 并发 + context + 测试  
- [ ] `net/http` 服务 + 中间件 + 配置 + slog  
- [ ] PostgreSQL CRUD + 迁移  
- [ ] Docker 运行起来  

**小项目：** 记事本 API（注册登录 + CRUD）。

### Phase 1 — Client API MVP（与社交闭环对齐）

- [ ] JWT 鉴权、错误码、request_id  
- [ ] 用户 / 关系 / 帖子 / Feed cursor  
- [ ] Redis 缓存与限流雏形  
- [ ] OpenAPI 契约导出给客户端  

**小项目：** 迷你社交 API（无推荐也可）。

### Phase 2 — AI 调用服务 MVP

- [ ] Provider 接口 + 一个上游实现  
- [ ] 同步 Chat  
- [ ] 流式 SSE  
- [ ] 用量日志 + 基础限流  
- [ ] 密钥与配置隔离  

**小项目：** `ai-gateway` 单机可演示流式聊天。

### Phase 3 — 打通 App 链路

- [ ] Client API AI 门面 + 会话表  
- [ ] gRPC/HTTP 内网调用 AI 服务  
- [ ] 取消传播与错误码映射  
- [ ] 移动端对接 SSE（或先用 Web 验证）  

### Phase 4 — 生产化

- [ ] 多模型路由与降级  
- [ ] Prompt 版本  
- [ ] 配额与成本看板  
- [ ] 内容安全  
- [ ] OTel 全链路  
- [ ] K8s 部署与压测  

### Phase 5 — 进阶（按需）

- [ ] Tool calling / Agent 状态机  
- [ ] RAG  
- [ ] 异步任务（长文档总结）  
- [ ] 多租户 / 企业级网关能力  

---

## 14. 推荐学习资源（类型，而非堆链接）

| 方向 | 学什么材料 |
|---|---|
| Go | 官方 Effective Go、Go Blog（context、concurrency）、自己读 `net/http` 源码关键路径 |
| API | HTTP API 设计指南；OpenAPI 规范 |
| 分布式基础 | 限流、熔断、重试、幂等、消息队列语义 |
| LLM 应用 | 官方 API 文档（Chat/Streaming/Tool）；Prompt 与安全基础；不要从重型 Agent 框架倒着学 |
| 观测 | OpenTelemetry 文档；Prometheus 直方图最佳实践 |
| 安全 | OWASP API Top 10；LLM 应用威胁模型入门 |

原则：**先官方协议与最小自研，再考虑 Lang* 类框架**。Go 生态里框架更碎，自研适配层往往更清晰。

---

## 15. 能力自检清单

### Go 与工程

- [ ] 能讲清 context 取消如何停掉下游 AI 请求  
- [ ] 能用 `-race` 跑过并发单测  
- [ ] 服务分层清晰，handler 不直接堆业务  

### Client API

- [ ] 有统一错误码与分页契约  
- [ ] 鉴权与资源归属校验完整  
- [ ] Feed/IM 至少一种实时或准实时通路跑通  
- [ ] 媒资预签名上传闭环  

### AI 服务

- [ ] Provider 可替换，不改业务代码  
- [ ] 流式 + 取消 + 心跳可用  
- [ ] 429/超时有退避与可观测  
- [ ] 用量可按用户聚合  
- [ ] Key 不进日志；有基础内容安全钩子  

### 联调

- [ ] App（或 curl/Web）能流式对话  
- [ ] 超额返回明确错误码  
- [ ] trace 能从网关跟到模型调用 span  

---

## 16. 常见坑（提前躲开）

1. **App 直连模型 Key** — 必漏；一律服务端。  
2. **流式中间件缓冲** — Gzip/部分代理会毁掉 SSE。  
3. **总超时套在流式上过短** — 首包与总时长要分开配。  
4. **超时重试导致重复 tool 副作用** — 工具执行要幂等或禁止自动重试。  
5. **把 AI 当普通 CRUD** — 必须按长请求与连接数重新做容量模型。  
6. **Prompt 与代码死绑** — 无法灰度与回滚。  
7. **日志打印全文对话** — 隐私与成本双爆。  
8. **微服务拆太早** — 先模块化单体：`client-api` + `ai-gateway` 两个进程足够 0→1。  

---

## 17. 与「五层移动端架构」的衔接

| 移动端层 | 对应后端知识 |
|---|---|
| L1 壳 | 深链进 AI 页；强更与登录门禁 |
| L2 AI 对话 Feature | 只消费 Client API 的会话/流式接口 |
| L3 Conversation/AI Domain | 本地消息投影、发送态；可复用 IM 壳 |
| L4 网络 | SSE/HTTP；不持有模型协议细节 |
| 后端 Client API | 用户鉴权、会话归属、配额入口 |
| 后端 AI 服务 | Provider、Prompt、流式、计量、安全 |

---

## 18. 下一步可沉淀的仓库产物

学的过程中建议在本工作区逐步落下：

```
notes/go-ai-and-client-service-learning.md   # 本文
experiments/go-client-api-skeleton/          # Phase 1 练习
experiments/go-ai-gateway-skeleton/          # Phase 2 练习
prompts/ai-gateway/                          # Prompt 模板样例
```

---

## 附录 A. 最小概念词汇表

| 词 | 含义 |
|---|---|
| TTFT | 首 token 时延 |
| TPM/RPM | 每分钟 token/请求限额 |
| Provider | 模型供应商适配实现 |
| BFF | Backend for Frontend，面向客户端的聚合 API |
| SSE | Server-Sent Events，单向服务端推流 |
| Tool Calling | 模型回调外部工具的协议化能力 |
| RAG | 检索增强生成 |
| Idempotency | 同一请求执行多次效果相同 |

## 附录 B. 两周突击课表（兼职可用）

| 天 | 内容 |
|---|---|
| D1–2 | Go HTTP 服务 + DB + JWT |
| D3–4 | Feed/帖子 API + OpenAPI |
| D5–6 | AI Provider 同步调用 + 日志用量 |
| D7–8 | 流式 SSE + 取消 |
| D9–10 | Client API 门面打通 + 错误码 |
| D11–12 | 限流配额 + 第二模型路由 |
| D13–14 | Docker 组成品演示 + 压测与复盘 |

---

**文档版本：** v1  
**维护：** 随实践里程碑勾选 §13 / §15，并把踩坑补进 §16。
