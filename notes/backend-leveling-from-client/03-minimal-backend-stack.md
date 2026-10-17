# 最小后端栈详解

> 对应练级文档 L0：「Web 框架 + 路由 / 配置 / 结构化日志 / 健康检查 / Docker」。  
> 目标：用**最少组件**跑起一个可观测、可部署的 HTTP 服务——这是从「会调 API」到「会提供 API」的第一块踏脚石。

下文以 **Go + net/http（或 Gin/Chi）** 为主示例；Kotlin/Spring、Node/Express 对照见文末。

---

## 0. 为什么是这五件，而不是一上来就上微服务？

客户端转后端常见误区：第一天就 Spring Cloud / K8s / 服务网格。

最小栈的本质是回答五个问题：

| 组件 | 回答的问题 | 客户端类比 |
|------|------------|------------|
| Web 框架 + 路由 | 请求怎么进来、谁处理？ | `URLSession` / Retrofit 的「反面」：你定义 path |
| 配置（环境变量） | 不同环境如何换 DB/密钥且不改代码？ | `Info.plist` / `BuildConfig` / `.xcconfig` |
| 结构化日志 | 线上出了事怎么定位？ | 你们现有的埋点/Crash 日志，但是服务端版 |
| `GET /healthz` | 进程活着吗？能否接流量？ | App 的「冷启动成功」信号，给探活用 |
| Docker | 别人机器 / CI / 线上怎么一键复现？ | 固定运行时，告别「我本地可以」 |

五件齐了，你才有资格谈数据库和鉴权；缺任何一件，后面都会返工。

---

## 1. Web 框架 + 路由

### 1.1 它在干什么？

把「一次 TCP 上的 HTTP 请求」变成「调用你的某个函数」，并约定：

- **Method + Path** → Handler  
- **中间件链**：日志 → 鉴权 → 业务（类似客户端拦截器 / OkHttp Interceptor）  
- **参数绑定**：path / query / header / body  
- **响应写出**：状态码 + JSON

### 1.2 路由要怎么设计才像后端？

```
GET    /healthz                 # 探活，见第 4 节
GET    /v1/bookmarks            # 列表
POST   /v1/bookmarks            # 创建
GET    /v1/bookmarks/{id}       # 详情
PATCH  /v1/bookmarks/{id}       # 部分更新
DELETE /v1/bookmarks/{id}       # 删除
```

原则（客户端出身尤其要守）：

1. **资源名词复数 + HTTP 动词**，少用 `/getUser`、`/doCreate` 这种 RPC 味 URL（内部 RPC 另说）。  
2. **版本放在路径或 Header**，一开始就假设会有老客户端。  
3. **Handler 保持薄**：只做协议转换；业务进 service。  
4. **统一错误 JSON**：例如 `{"code":"...","message":"...","request_id":"..."}`，别有的接口返回字符串、有的返回 HTML。

### 1.3 最小可运行示意（Go 标准库）

```go
package main

import (
	"encoding/json"
	"log/slog"
	"net/http"
	"os"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", handleHealthz)
	mux.HandleFunc("GET /v1/bookmarks", handleListBookmarks)
	mux.HandleFunc("POST /v1/bookmarks", handleCreateBookmark)

	addr := ":" + envOr("PORT", "8080")
	slog.Info("listening", "addr", addr)
	if err := http.ListenAndServe(addr, withRequestID(mux)); err != nil {
		slog.Error("server stopped", "err", err)
		os.Exit(1)
	}
}

func handleHealthz(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
```

框架选型直觉：

| 栈 | 入门选择 | 说明 |
|----|----------|------|
| Go | `net/http` → Chi / Gin | 先标准库理解本质，再加路由糖 |
| Kotlin | Ktor 或 Spring Web | 要就业面选 Spring；要轻量选 Ktor |
| Node | Express / Fastify | 迁移快；注意类型与错误处理纪律 |

### 1.4 中间件：把客户端「拦截器」搬到服务端

每个请求建议固定经过：

1. 生成 / 透传 `X-Request-ID`  
2. 记录 method、path、status、latency  
3. recover panic → 500（不要把栈直接回给客户端）  
4. （稍后）鉴权、限流  

这和 OkHttp / Alamofire 拦截器同构，只是你现在站在服务端。

---

## 2. 配置（环境变量）

### 2.1 为什么用环境变量，而不是写死在代码里？

因为**同一份产物**要跑在：本地、CI、预发、生产。  
配置变、镜像不变——这是 12-Factor App 的核心，也是 Docker/K8s 的默认约定。

客户端类比：

- Debug / Release 不同 `BASE_URL`  
- 证书、API Key 不进仓库  

后端同理：`DATABASE_URL`、`REDIS_ADDR`、`JWT_SECRET` **永不提交 git**。

### 2.2 最小配置集（L0 够用）

```bash
PORT=8080
APP_ENV=dev                    # dev | staging | prod
LOG_LEVEL=info                 # debug | info | warn | error
DATABASE_URL=postgres://...    # 有 DB 再加
JWT_SECRET=change-me           # 至少 32 字节随机串
```

读取约定：

1. **必填项缺失 → 启动失败**（fail fast），不要默默用错误默认值上线。  
2. **有合理默认的才给默认**（如 `PORT=8080`）。  
3. 用一小层 `config` 包集中解析，业务代码不直接 `os.Getenv` 散落。  
4. 本地可用 `.env`（dotenv），但**生产以编排系统注入为准**；`.env` 进 `.gitignore`。

### 2.3 配置与密钥分层

| 类型 | 例子 | 存放 |
|------|------|------|
| 非敏感 | `PORT`、`LOG_LEVEL`、`APP_ENV` | 环境变量 / ConfigMap |
| 敏感 | DB 密码、JWT 密钥、第三方 Token | Secret / 密钥管理服务 |
| 功能开关 | 降级开关、灰度比例 | 配置中心（L4 再上） |

L0 先把「环境变量 + `.env.example`（无密钥）」做漂亮即可。

### 2.4 `.env.example` 模板

```bash
# copy to .env and fill in; never commit .env
PORT=8080
APP_ENV=dev
LOG_LEVEL=debug
# DATABASE_URL=postgres://user:pass@localhost:5432/app?sslmode=disable
# JWT_SECRET=
```

同事 / 未来的你：`cp .env.example .env` 就能开工。

---

## 3. 结构化日志

### 3.1 什么叫「结构化」？

不要：

```
User 42 failed something somehow
```

而要（JSON 一行一条，方便 ELK / Loki 检索）：

```json
{"time":"2026-10-17T12:00:00Z","level":"ERROR","msg":"create bookmark failed","request_id":"a1b2","user_id":42,"err":"unique violation","path":"/v1/bookmarks"}
```

字段是键值对；**msg 是事件名，细节进字段**。

### 3.2 每个请求至少要有的字段

| 字段 | 用途 |
|------|------|
| `request_id` | 贯穿一次调用；客户端也可上传，便于端到端排查 |
| `method` / `path` | 定位接口 |
| `status` / `latency_ms` | 性能与错误率 |
| `user_id`（若已鉴权） | 用户维排查 |
| `err` | 失败原因（注意脱敏） |

### 3.3 日志级别怎么用

| Level | 何时 |
|-------|------|
| DEBUG | 本地开发、临时排查（生产默认关闭） |
| INFO | 启动、关键业务节点、请求完成摘要 |
| WARN | 可恢复异常：重试成功、降级命中、接近限流 |
| ERROR | 请求失败需人关注：未处理异常、依赖失败 |

禁忌：

- 打密码、Token、身份证号全文  
- 在热路径 `DEBUG` 打印巨大 body（拖垮磁盘与性能）  
- 用日志代替指标（QPS/P99 仍应有 metrics，L2+ 再补）

### 3.4 和客户端埋点的关系

你熟悉的「事件名 + 属性」就是结构化日志。差别是：

- 客户端日志常抽样、可丢；  
- 服务端错误日志是排障主路径，**不能丢关键错误**，但要控制体积。

给每个请求注入 `request_id`，并在错误响应里返回（或至少在响应头 `X-Request-ID`），客户端报 bug 时把这个 ID 丢给你——效率质变。

---

## 4. 健康检查 `GET /healthz`

### 4.1 它解决什么问题？

Docker / K8s / 负载均衡需要知道：**这个进程能不能接流量？**

没有健康检查时：

- 进程卡住了还在接收流量 → 用户全员超时  
- 滚动发布时旧实例没排空就杀 → 请求被掐断  

客户端类比：应用「启动完成」才展示首页；探活是给**基础设施**看的心跳。

### 4.2 两种检查（名字因平台而异）

| 类型 | 含义 | 典型实现 |
|------|------|----------|
| **Liveness** | 进程是否假死，该不该重启 | `/healthz` 只确认进程能响应 HTTP |
| **Readiness** | 是否准备好接流量 | `/readyz` 检查 DB/Redis 是否连通 |

L0 可以先只做一个：

```http
GET /healthz HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json

{"status":"ok"}
```

进阶（L1+）拆成：

```http
GET /healthz   → 永远尽快 200（别查 DB，避免探活把自己打死）
GET /readyz    → DB ping 失败则 503，从负载均衡摘流
```

### 4.3 实现注意点

1. **探活接口要轻**：不要在 `/healthz` 里跑复杂业务。  
2. **不要鉴权**：探活来自集群内部或本机。  
3. **依赖挂了**：用 readiness 摘流，而不是让 liveness 疯狂重启（重启解决不了 DB 宕机）。  
4. 返回 JSON 便于人眼和脚本；K8s 只关心状态码也可以。

### 4.4 和客户端自测

```bash
curl -sS http://localhost:8080/healthz
# {"status":"ok"}
```

容器起来后，**先 curl healthz，再测业务接口**——这是后端日常肌肉记忆。

---

## 5. Docker 一键跑起来

### 5.1 为什么 L0 就要 Docker？

- 统一 Go/JDK/Node 版本，消灭「我机器可以」  
- CI 与本地同一构建路径  
- 为以后 Compose 加 MySQL/Redis 打地基  
- 专栏与行业实践都把容器当默认交付单元  

客户端类比：固定 Xcode / AGP 版本的 CI 镜像——只是后端把整个服务打进镜像。

### 5.2 最小 `Dockerfile`（Go 多阶段构建）

```dockerfile
# build
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/server ./cmd/server

# run
FROM gcr.io/distroless/static-debian12
ENV PORT=8080
EXPOSE 8080
COPY --from=build /out/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

要点：

- **多阶段**：最终镜像不含编译器与源码，体积小、攻击面小  
- **非 root 用户**跑进程  
- 用环境变量注入配置，镜像内不烤密钥  

### 5.3 `docker-compose.yml`（本地一键）

```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      PORT: "8080"
      APP_ENV: dev
      LOG_LEVEL: debug
      # DATABASE_URL: postgres://app:app@db:5432/app?sslmode=disable
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8080/healthz"]
      interval: 5s
      timeout: 3s
      retries: 5
    # depends_on:
    #   db:
    #     condition: service_healthy
```

一键：

```bash
docker compose up --build
curl -sS localhost:8080/healthz
```

有 DB 时再加 `db` 服务与 `depends_on`；L0 无 DB 也可以先只跑 API。

### 5.4 `.dockerignore`

```
.git
.env
*.md
tmp/
bin/
```

避免把密钥和无关文件打进构建上下文。

### 5.5 验收清单（做完才算「一键跑起来」）

- [ ] 新同事只装 Docker，不装语言工具链，也能 `compose up` 通 healthz  
- [ ] 改 `PORT` / `LOG_LEVEL` 只改环境变量，不改代码重建逻辑  
- [ ] 日志是 JSON（或明确的 key=value），且带 `request_id`  
- [ ] `README` 写清：启动命令、探活 URL、示例 curl  

---

## 6. 五件如何串成一条请求链路

```
客户端请求
    ↓
Docker 容器内进程监听 :8080
    ↓
路由匹配 METHOD + PATH
    ↓
中间件：request_id → 日志 → （鉴权）
    ↓
Handler → Service →（DB）
    ↓
结构化日志一行（status + latency）
    ↓
JSON 响应；错误含稳定 code
```

基础设施侧：

```
编排器定期 GET /healthz（或 /readyz）
    ↓
失败 → 摘流 / 重启
成功 → 继续接流量
```

---

## 7. 一天动手任务（建议照做）

1. 选语言，建 repo：`cmd/server` + `internal/config` + `internal/httpapi`  
2. 实现 `/healthz` + 一个 `GET /v1/ping`  
3. 配置从环境变量读 `PORT`、`LOG_LEVEL`  
4. 请求日志带 `request_id`  
5. 写 Dockerfile + compose，卸掉本机直接进程，只用容器访问  
6. 用手机/模拟器或 `curl` 打通；故意停掉容器，观察 healthcheck 失败  

验收口令：

> 「我可以在任意一台装了 Docker 的机器上，不改代码，只改环境变量，把服务跑起来，并证明它活着。」

---

## 8. 其他语言最小对照

### Spring Boot（Kotlin/Java）

- 路由：`@RestController` + `@GetMapping`  
- 配置：`application.yml` + env 覆盖（`SERVER_PORT`）  
- 日志：Logback JSON encoder（或 Logstash encoder）  
- 健康：Spring Actuator `/actuator/health`（可当 healthz）  
- Docker：Buildpacks 或多阶段 JRE 镜像  

### Node（Fastify/Express）

- 路由：`app.get('/healthz', ...)`  
- 配置：`process.env` + `zod`/`envalid` 校验  
- 日志：`pino`（天然 JSON）  
- 健康：独立路由；K8s 分 liveness/readiness  
- Docker：`node:22-alpine` 多阶段，`NODE_ENV=production`  

---

## 9. 和练级文档的衔接

做完最小栈 → 回到 [01-leveling-guide.md](./01-leveling-guide.md) 实践项目 A：

- 在最小栈上加 JWT 鉴权与 CRUD  
- 日志贯穿 `user_id` + `request_id`  
- Compose 中加入 Postgres  
- 用客户端 App 对接，并验证弱网重试与错误码  

下一篇可继续拆：[02-interview-qa.md](./02-interview-qa.md) 中的超时预算、连接池、网关探活相关题，都会回到「健康检查 + 配置 + 日志」这三块基本功。
