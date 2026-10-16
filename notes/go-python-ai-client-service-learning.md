# Go 客户端服务 + Go/Python AI 服务：联合知识体系与学习指南

> 适用：社交 / 内容类 App 后端——**Client API 用 Go**；**AI 链路以 Go 网关为主**，在确有必要时 **叠加 Python 做 AI 处理**。  
> 前序纯 Go 版：[go-ai-and-client-service-learning.md](./go-ai-and-client-service-learning.md)  
> 原则：**不是默认双语言**；Python 只承担 Go 不划算或生态明显更强的那一段。  
> 交互版：[Go+Python AI与客户端服务学习](https://cursor.com/canvas/store-3f310d1e-80b7-486b-8c4a-9718b9ab9225/a314172c-5b60-40e1-aada-78029b8c16a1)。

---

## 0. 结论先讲清

| 问题 | 答案 |
|---|---|
| 客户端服务要不要 Python？ | **一般不要**。鉴权、Feed、IM、媒资、BFF 用 Go。 |
| AI 服务要不要 Python？ | **看负载类型**。纯「调模型 API + 流式转发 + 限流计量」Go 足够；涉及重 ML/数据/Agent 生态时再加 Python。 |
| 怎么合？ | **Go AI Gateway 对外统一出口**；Python 作 **Worker / Skill Runtime / Offline Pipeline**，经 gRPC/HTTP/MQ 调用。 |
| 学什么？ | Go 全栈后端 + Python AI 处理专项 + **服务间契约与编排**（比再学一门框架更重要）。 |

```
App / Web
   │ HTTPS / WSS / SSE
   ▼
┌──────────────── Client API（Go）────────────────┐
│ 鉴权 · Feed · 关系 · IM · 媒资 · AI 会话门面     │
└───────────────┬──────────────────┬──────────────┘
                │                  │
                ▼                  ▼
        业务领域服务(Go)    ┌──── AI Gateway（Go）────┐
                            │ 鉴权内网 · 配额 · 路由   │
                            │ 流式汇聚 · 审计 · 降级   │
                            └─────┬──────────┬───────┘
                     直接调模型 API │          │ 需要时
                                  ▼          ▼
                            OpenAI/国产等   ┌─────────────────────┐
                                           │ AI Worker（Python）  │
                                           │ RAG · 解析 · Agent   │
                                           │ 评测 · 批处理 · 微调辅 │
                                           └──────────┬──────────┘
                                                      ▼
                                              向量库 / 文件 / 模型 API
```

---

## 1. 何时必须 / 建议叠加 Python

### 1.1 建议用 Python 的信号（满足 2 条以上再引入）

| 场景 | 原因 |
|---|---|
| **RAG 全链路**（切分、嵌入批跑、混合检索、重排） | `llama-index` / 自研 pipeline、向量生态、实验迭代快 |
| **文档/多模态预处理**（PDF/Office/ASR 后处理、OCR 编排） | Python 库成熟；Go 绑定少 |
| **Agent / Tool 密集编排**（多步、动态规划、评测 harness） | 研究与示例多在 Python；可先 PoC 再固化 |
| **模型侧本地推理或训练辅助**（LoRA 数据、评测集、embedding 小模型） | PyTorch / transformers 基本盘 |
| **算法同学主产出在 notebook** | 降低翻译成本；用服务边界收口，而不是把 notebook 暴露给 App |

### 1.2 继续留在 Go 的（不要迁去 Python）

| 场景 | 原因 |
|---|---|
| 用户 JWT、业务 CRUD、Feed/IM | 延迟与工程一致性 |
| API Key 保管、全局限流、配额扣减、统一错误码 | 网关职责；单点策略 |
| 对 App 的 SSE/WS 长连接出口 | Go 并发与连接模型更稳 |
| 仅转发 Chat Completions 流式 | 自研几十～几百行适配即可 |
| 计费、审计、租户隔离 | 与业务账号体系同进程/同语言更易正确 |

### 1.3 反模式

1. App → Python Flask 直出业务 + 模型（密钥与业务耦合、难扩）。  
2. 为「将来可能用到 RAG」一上来 Go+Python+K8s+向量库全家桶。  
3. Python 与 Go 双写同一套用户鉴权。  
4. 用 Python 扛所有高 QPS 同步 API。  

**默认路径：** 先 Go Client API + Go AI Gateway；出现 §1.1 场景再加 `ai-worker-py`。

---

## 2. 推荐进程拆分（0→1 → 生产）

| 阶段 | 进程 | 说明 |
|---|---|---|
| 0→1 最小 | `client-api`（Go）+ `ai-gateway`（Go） | 无 Python |
| 出现重 AI | + `ai-worker`（Python） | 网关按 `scene` 路由到 worker 或直连模型 |
| 规模化 | Client / Gateway / Worker 独立扩缩 | Worker 可再拆 rag-worker / agent-worker |

**场景路由示例（放在 Go Gateway）：**

```text
scene=chat.simple     → Go 直调模型（流式）
scene=chat.rag        → gRPC → Python RAG → 再调模型（或 Python 内调）
scene=doc.summarize   → MQ → Python 异步任务 → 回调/写库
scene=agent.tools     → Python Agent Runtime（步数/费用熔断由 Gateway 配额兜底）
```

---

## 3. 知识地图（Go + Python 联合）

```
A. Go 后端基础（Client + Gateway）
B. 客户端服务域知识（业务 API）
C. Go AI Gateway（流式、限流、多模型）
D. Python AI Worker（何时上、学什么）
E. 双语言协作契约（gRPC/HTTP/MQ/事件）
F. 数据与中间件（共享）
G. 安全 · 观测 · 部署（跨语言统一）
H. 学习路径与里程碑
I. 自检清单与坑
```

---

## 4. A–B. Go 客户端服务（摘要，细则见纯 Go 文档）

必须掌握（不因引入 Python 而减少）：

- Go 并发、`context`、error、模块与测试  
- HTTP/JSON、OpenAPI、cursor 分页、幂等、JWT  
- Feed / 关系 / IM / 媒资 / AI 会话门面  
- WebSocket；对 App 的 SSE 透传  
- PostgreSQL/MySQL + Redis；统一错误码与 `request_id`  

**Client API 对 AI 只做：** 鉴权、会话归属、配额预检、把流式结果转给 App。  
**不在 Client API 里：** 调向量库、跑 Agent 循环、解析 PDF。

详见：[go-ai-and-client-service-learning.md](./go-ai-and-client-service-learning.md) §3–§5。

---

## 5. C. Go AI Gateway——永远是对外「AI 控制面」

即使有 Python，Gateway 仍建议用 Go，并保持这些职责：

| 职责 | 说明 |
|---|---|
| 内网鉴权 | 只接受 Client API / 服务账号 |
| 路由 | `scene` → 直连模型 / Python worker / 异步队列 |
| 流式汇聚 | 把 worker 或供应商的流归一成对内 `StreamEvent` |
| 限流配额 | 用户/租户 TPM、并发 in-flight |
| 超时取消 | 首包超时 vs 总超时；ctx 取消传到 Python |
| 降级 | worker 挂了可降级到「无 RAG 直答」或友好错误 |
| 审计计量 | 统一写 usage；Python 回传 token 也在此汇总 |
| Key 管理 | 模型 Key 可只放 Gateway，或 Worker 独立 Key 但仍经 Gateway 授权调用 |

Go Gateway 内部接口保持稳定（与纯 Go 版相同）：

```go
type ChatClient interface {
    Chat(ctx context.Context, req ChatRequest) (*ChatResponse, error)
    ChatStream(ctx context.Context, req ChatRequest) (<-chan StreamEvent, error)
}
```

Python Worker 是其中一种 `ChatClient` 适配器实现（via gRPC），对上业务无感。

---

## 6. D. Python AI Worker——要学的知识（展开）

### 6.1 语言与工程（服务化，不是 notebook 思维）

| 主题 | 学到什么程度 |
|---|---|
| Python 3.11+ 类型标注 | `typing`、Pydantic 模型与 Gateway 契约对齐 |
| 异步 | `asyncio`、httpx/aiohttp；能流式读上游 SSE |
| 打包 | `uv` / `poetry`；锁文件；多阶段镜像 |
| 项目结构 | `app/api` · `app/services` · `app/providers` · `app/pipelines` |
| 测试 | pytest；核心 pipeline 可离线假数据跑通 |
| 配置 | 环境变量；**禁止**把 Key 写进仓库 |

### 6.2 Web / RPC 服务形态

| 选型 | 用途 |
|---|---|
| **FastAPI** + Uvicorn | HTTP/SSE 对内接口（简单场景） |
| **gRPC（grpcio）** | 与 Go Gateway 主路径（推荐） |
| 任务队列 | Celery / ARQ / RQ + Redis；或听 Kafka/NATS |

要学：生命周期、优雅退出、健康检查、`Cancellation` 与超时（和 Go ctx 对齐）。

### 6.3 LLM 应用层（Python 强项）

| 主题 | 具体技能 |
|---|---|
| Provider SDK | OpenAI 官方 SDK、兼容网关；流式 iterator |
| Prompt | 模板、结构化输出、防注入；版本可用文件或 CMS |
| RAG | 加载 → 切分 → embed → 检索 → 重排 → 组装上下文 → 生成 |
| 解析 | PDF/HTML/Markdown；表格与元数据 |
| Agent | tool schema、多步循环、最大步数/费用熔断（与 Gateway 配额双保险） |
| Eval | 黄金集、回归；上线前抽检幻觉与拒答 |

框架态度：

- **先会裸调 SDK + 自研薄封装**，再按需引入 LangChain / LlamaIndex / 其它。  
- 选框架的标准：能否清晰画出「哪一步可替换」，而不是全家桶绑死。

### 6.4 数据与 AI 设施

| 组件 | 用途 |
|---|---|
| 向量库 | pgvector / Qdrant / Milvus（选一吃透） |
| 对象存储 | 文档与切片产物 |
| Embedding 服务 | 自建或 API；批量与限速 |
| 特征/文档元数据库 | 与业务库分离或同实例不同 schema |

### 6.5 Python Worker **不要做**的事

- 不校验用户 JWT（信任 Gateway 传入的 `user_id`，并校验服务间 token）  
- 不对公网暴露  
- 不直接写 Feed/IM 业务表（结果回 Gateway / 写 AI 专用表）  
- 不在请求线程里做长时间训练  

---

## 7. E. 双语言协作——这是「合在一起」的核心课

### 7.1 契约优先（一份 proto / OpenAPI 两边生成）

推荐 **gRPC + proto** 作为 Go ↔ Python 主契约：

```protobuf
service AIWorker {
  rpc Retrieve(RetrieveRequest) returns (RetrieveResponse);
  rpc Chat(ChatRequest) returns (ChatResponse);
  rpc ChatStream(ChatRequest) returns (stream StreamEvent);
  rpc RunTask(TaskRequest) returns (TaskHandle); // 异步
}

message StreamEvent {
  string type = 1; // delta | usage | error | done
  string delta = 2;
  Usage usage = 3;
  string error_code = 4;
}
```

字段约定：

- `request_id` / `trace_id` 全链路必带  
- `user_id` 由 Gateway 注入（已鉴权）  
- `scene` / `prompt_key` / `prompt_version`  
- `deadline_unix_ms` 或依赖 gRPC deadline  

### 7.2 同步流式 vs 异步任务

| 模式 | 适合 | 协作方式 |
|---|---|---|
| 同步流式 | 对话、带 RAG 的短问答 | gRPC stream；Gateway 再转 App SSE |
| 同步请求-响应 | 检索、短总结 | unary RPC |
| 异步任务 | 长文档、批量嵌入、报表 | MQ + 状态表；App 轮询或推送 |

### 7.3 取消与超时（必对齐）

```
用户点停止
  → App 断 SSE
  → Client API cancel
  → Go Gateway cancel ctx / gRPC cancel
  → Python 收到 cancelled，停止读上游模型
```

Python 侧要学：在 async for 中检测 cancel；释放 httpx stream；不要吞掉 `CancelledError`。

### 7.4 错误码映射

| Worker 错误 | Gateway → App |
|---|---|
| 检索空 | 可降级直答或 `AI_RAG_EMPTY` |
| 上游 429 | `AI_RATE_LIMITED` + Retry-After |
| 超时 | `AI_UPSTREAM_TIMEOUT` |
| 内容安全 | `AI_CONTENT_BLOCKED` |
| 内部未捕获 | `AI_INTERNAL`（日志含 request_id） |

### 7.5 观测统一

- Trace：OTel 在 Go 与 Python 都开，**同一 `trace_id`**  
- Metrics：`ai_ttft`、`worker_stage_latency{stage=retrieve|rerank|generate}`  
- 日志：结构化 JSON；对话正文默认不落或脱敏采样  

### 7.6 本地开发组合

```bash
docker compose up  # postgres redis
go run ./cmd/client-api
go run ./cmd/ai-gateway
uv run uvicorn app.main:app  # 或 grpc 服务
```

用 `buf`/`protoc` 生成 Go 与 Python stub；CI 校验 proto 兼容。

---

## 8. F. 共享数据与所有权

| 数据 | 所有者 | 谁写 |
|---|---|---|
| 用户/关系/Feed/IM | 业务库 | 仅 Go 业务服务 |
| AI 会话与消息摘要 | AI 库或业务库 AI schema | Gateway（主），Worker 可写 chunk/引用 |
| 用量与配额 | Gateway | 仅 Go |
| 向量与文档切片 | AI 专用存储 | 主要 Python |
| Prompt 版本 | 配置中心或 DB | 发布流程；运行期两边只读 |

**所有权不清是双语言项目第一失败点**——表必须有唯一写者。

---

## 9. G. 安全 · 部署（跨语言）

### 安全

- Python 不暴露公网；仅 ClusterIP / 内网  
- 服务间 mTLS 或签发短效 service JWT  
- Worker 若有 HTTP 工具：SSRF 白名单  
- 依赖扫描：`govulncheck` + `pip-audit`/`uv audit`  
- 模型 Key：Vault/KMS；按环境隔离  

### 部署

| 服务 | 扩缩参考 |
|---|---|
| client-api | QPS、CPU |
| ai-gateway | 并发流式连接数、CPU |
| ai-worker | 队列深度、GPU/CPU、上游 RPM |

镜像：Go 多阶段小镜像；Python 用瘦基础镜像 + 锁定依赖；避免把 torch 塞进每个无 GPU 任务的镜像（按 worker 类型拆镜像）。

网关超时：对 AI 路由单独加大；注意 Ingress 对 SSE/gRPC 的支持。

---

## 10. H. 联合学习路径（含 Python 决策点）

### Phase 0 — Go 基本功

与纯 Go 版相同：HTTP + DB + JWT + Docker。

### Phase 1 — Client API MVP（纯 Go）

用户 / 帖子 / Feed / OpenAPI。**无 Python。**

### Phase 2 — Go AI Gateway MVP

同步 Chat → SSE → 限流用量。**无 Python。**  
打通：`App → Client API → Gateway → 模型`。

### Phase 3 — 决策关卡（是否引入 Python）

用真实需求打分：

- [ ] 需要文档解析 / RAG 吗？  
- [ ] 需要多步 Agent 且算法同学用 Python 交付吗？  
- [ ] 仅 Chat 补全是否已满足产品？  

**否 → 停在 Phase 2 加固生产化。**  
**是 → 进入 Phase 4。**

### Phase 4 — Python Worker 最小闭环

- [ ] FastAPI 或 gRPC 服务骨架 + 类型化请求  
- [ ] 一个 `Retrieve` 或 `Chat with context`  
- [ ] Gateway 增加 `PythonWorkerClient` 适配器  
- [ ] 流式或 unary 打通 + 取消传播  
- [ ] 统一 trace_id 与错误码  

**练习项目：** 「对我上传的 3 篇笔记做问答」（上传走 Go 媒资，检索生成走 Python）。

### Phase 5 — 生产化双语言

- [ ] 配额与费用熔断（Gateway）  
- [ ] Worker 异步长任务  
- [ ] 内容安全钩子  
- [ ] 压测：Gateway 连接数 + Worker 队列  
- [ ] Prompt/RAG 版本与评测回归  

### Phase 6 — 进阶

- [ ] 重排模型、混合检索  
- [ ] Tool calling 白名单与人工确认  
- [ ] 多 worker 类型（rag / agent / etl）  
- [ ] 可选：embedding 自建服务  

---

## 11. 两周 + 加一周期课表（兼职）

| 周期 | 内容 |
|---|---|
| 第 1–2 周 | 纯 Go：Client API + AI Gateway 流式聊天（同纯 Go 文档附录 B） |
| 第 3 周 | **仅当需要**：Python Worker + proto 契约 + Gateway 路由 `scene=rag` |
| 之后 | 观测、配额、评测、部署拆分 |

---

## 12. 能力自检（联合版）

### 架构决策

- [ ] 能说清「当前产品为什么要 / 不要 Python」  
- [ ] 画出三进程图与数据所有权  
- [ ] scene 路由表写得出来  

### Go

- [ ] Client API 与 Gateway 职责不混  
- [ ] 流式出口与配额在 Go  
- [ ] 能把 gRPC cancel 连到用户停止  

### Python

- [ ] Worker 无公网、无用户 JWT 逻辑  
- [ ] 至少一个 RAG 或解析 pipeline 可测  
- [ ] 取消与超时不泄露僵尸上游请求  

### 协作

- [ ] proto/OpenAPI 单源生成两边代码  
- [ ] 同一 trace 跨 Go/Python  
- [ ] 错误码到 App 可理解、可埋点  

---

## 13. 常见坑（双语言加倍）

1. **过早引入 Python** — 只有 Chat 却养两套运行时。  
2. **Python 对公网** — 变成第二套未加固 API。  
3. **两处扣配额** — 只在 Gateway 扣；Worker 只上报。  
4. **契约口口相传** — 必须 proto/OpenAPI + CI。  
5. **取消断一层** — 用户停了模型还在跑，烧钱。  
6. **日志双份全文** — 隐私与存储爆炸。  
7. **框架套娃** — Gateway 一层 Agent、Worker 又一层 Agent。  
8. **共享业务表乱写** — Worker 严禁直接改 Feed/IM。  

---

## 14. 与移动端五层架构的衔接

| 移动端 | 后端 |
|---|---|
| L2 AI 对话 UI | 只打 Client API |
| L3 会话 Domain | 本地投影发送态；不感知 Python |
| Client API（Go） | 用户态与 SSE 门面 |
| AI Gateway（Go） | AI 控制面 |
| AI Worker（Python） | 可选智能面（RAG/Agent/解析） |

---

## 15. 仓库内建议沉淀

```
notes/go-python-ai-client-service-learning.md  # 本文（联合版）
notes/go-ai-and-client-service-learning.md     # 纯 Go 版（仍保留）
experiments/go-client-api-skeleton/
experiments/go-ai-gateway-skeleton/
experiments/py-ai-worker-skeleton/             # Phase 4 再加
api/proto/ai/v1/worker.proto                   # 双语言契约
```

---

## 附录 A. 决策流程图（文字版）

```
要做 AI 能力？
  ├─ 否 → 只做 Go Client API
  └─ 是 → 做 Go AI Gateway
           ├─ 仅 Chat/流式/限流 → 停（纯 Go）
           └─ 需要 RAG / 重解析 / Agent 密集 / 算法 Python 交付
                    → 加 Python Worker，Gateway 统一出口
```

## 附录 B. 词汇补遗

| 词 | 含义 |
|---|---|
| AI Gateway | AI 控制面：路由、配额、流式、审计 |
| AI Worker | AI 智能面：RAG/Agent/解析等重处理 |
| Scene | 调用场景键，决定路由与 Prompt |
| Control plane / Data plane | 控制策略 vs 实际模型与检索算力 |

## 附录 C. 文档关系

| 文档 | 定位 |
|---|---|
| 本文 | **主推荐**：Go + 按需 Python 的完整学习知识 |
| [纯 Go 版](./go-ai-and-client-service-learning.md) | Gateway/Client 细节更深的补充教材 |
| 社交移动端架构笔记 | App 五层如何对接上述后端 |

---

**文档版本：** v1（Go+Python 联合）  
**维护：** Phase 3 决策关卡之后，把真实 `scene` 路由表与仓库路径补进本文。
