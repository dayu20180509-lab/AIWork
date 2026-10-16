# Go + 按需 Python：AI / 客户端服务学习入口

**主文档（推荐）：**

→ [notes/go-python-ai-client-service-learning.md](../../notes/go-python-ai-client-service-learning.md)

**纯 Go 补充（Gateway/Client 细节）：**

→ [notes/go-ai-and-client-service-learning.md](../../notes/go-ai-and-client-service-learning.md)

## 核心结论

- **Client API：Go**（鉴权、Feed、IM、媒资、AI 门面）
- **AI Gateway：Go**（配额、流式出口、路由、审计）——始终是 AI 控制面
- **Python Worker：按需**（RAG、文档解析、Agent、算法交付）——不默认引入
- App 只打 Client API；**不直连模型，不直连 Python**

## 引入 Python 的关卡

先完成纯 Go 的 Client + Gateway 流式聊天；仅当需要 RAG / 重解析 / 密集 Agent 时再加 `ai-worker`。
