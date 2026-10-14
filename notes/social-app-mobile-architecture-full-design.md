# 社交 App 移动端技术架构完整设计方案（0→1）

> 目标：从 0 到 1 交付一款双端原生社交 App 的可演进客户端架构。  
> 前提：产品形态为「关注关系 + Feed 消费 + 创作发布 + 私信」；AI 对话为可选增值，非一期核心。  
> 关联：分层评审见 [social-app-mobile-architecture.md](./social-app-mobile-architecture.md)。  
> 交互版总览：[社交 App 移动端完整设计方案](https://cursor.com/canvas/store-3f310d1e-80b7-486b-8c4a-9718b9ab9225/c6cd405e-d3f5-4195-8cf8-6c4cee2c674f)。

---

## 1. 设计目标与边界

### 1.1 目标

| 目标 | 度量 |
|---|---|
| 快速验证社交闭环 | 关注 → 消费 → 互动 → 私信 → 再创作 |
| 结构可演进 | 业务可增删，不推翻网络/存储/同步骨架 |
| 质量可观测 | 启动、首屏、发布成功率、IM 送达、崩溃有日报 |
| 双端体验一致 | 契约、错误码、埋点、视觉基线对齐 |
| 为二期复用留口 | 领域 API 稳定后可接入 RN 或 KMP（择一） |

### 1.2 非目标（一期明确不做）

- 不全量自研音视频通话 / 直播
- 不上复杂推荐中台（可用服务端简单排序）
- 不并行推进 RN + KMP + SwiftUI 三线
- 不追求极致离线优先（先正确同步，再增强离线）
- 不在客户端做重型内容审核（审核在服务端）

### 1.3 一期产品范围（MVP）

**必做**

1. 手机号 / 一键登录、会话保持、基础资料
2. 关注 / 粉丝 / 个人主页
3. 关注流 Feed（图文为主）：浏览、点赞、评论、分享外链
4. 创作发布：图文（多图）、基础裁剪压缩
5. 私信 IM：单聊、会话列表、未读、推送拉起
6. 系统推送、深链、基础分享
7. 崩溃监控、核心漏斗埋点、远程配置开关

**可后置**

- 短视频发布与沉浸浏览
- 群聊、已读回执精细化、输入态
- AI 对话助手
- 复杂话题 / 搜索 / 推荐双列

---

## 2. 总体架构

### 2.1 分层（依赖自上而下）

```
┌──────────────────────────────────────────────────────────────┐
│ L1 应用壳 AppShell                                           │
│    启动、Tab、路由、深链、登录门禁、全局 UI（Toast/弹窗）        │
├──────────────────────────────────────────────────────────────┤
│ L2 业务模块 Features                                         │
│    feed · profile · create · im · auth-ui · (ai 可选)         │
├──────────────────────────────────────────────────────────────┤
│ L3 领域服务 Domain                                           │
│    AccountSession · Relation · FeedSync · Conversation       │
│    MediaPipeline · PushRouter · ConfigGate                   │
├──────────────────────────────────────────────────────────────┤
│ L4 平台能力 Platform                                         │
│    Net(HTTP/WS) · Storage · Image/VideoKit · PushChannel     │
│    Tracker · SecureStore · Device · Permission               │
├──────────────────────────────────────────────────────────────┤
│ L5 工程基建 Infra                                            │
│    UIKit · DI/Module · Logger · Crash · Perf · Build/CI      │
└──────────────────────────────────────────────────────────────┘

※ 跨端 CrossPlatform = 二期策略（不进入依赖栈）
```

### 2.2 依赖规则（强制）

1. **单向依赖**：L1→L2→L3→L4→L5；禁止反向。
2. **业务横向隔离**：`feed` 不得直接 import `im` 实现；跨模块只走 L3 接口或壳层路由。
3. **领域可持有私有态**：Feed/IM 可有私有 DB 与同步引擎，对外只暴露 Domain API。
4. **平台无业务**：L4 禁止依赖任何 Feature / Domain 业务模型（DTO 除外可放 `contracts`）。
5. **契约优先**：双端共享 OpenAPI / Proto、错误码表、埋点字典、深链 schema。

### 2.3 五层依赖怎么设计（详解）

> 分层的本质不是画框，而是规定：**谁可以 import 谁、状态存在哪、变化时改哪一层**。  
> 交互版：[五层依赖设计详解](https://cursor.com/canvas/store-3f310d1e-80b7-486b-8c4a-9718b9ab9225/4325443b-7f7d-46b8-b3c9-c71e8c588686)。

#### 2.3.1 设计原则

| 原则 | 含义 |
|---|---|
| **依赖单向、稳定向下** | 上层可依赖下层；下层改动不应倒逼上层大改；下层禁止感知上层业务 |
| **按变化原因拆层** | 因「产品交互」变的进 L2；因「业务规则/同步」变的进 L3；因「技术实现」变的进 L4/L5 |
| **越向下越可复用、越无产品语义** | L5 换崩溃 SDK 不影响 Feed；L2 改评论 UI 不应改网络库 |
| **允许垂直切片穿透存储** | Feed/IM 的 Domain 可直接用 L4 Storage，但对外仍只暴露 Domain API |
| **跨层只经公开接口** | Feature 不 new OkHttp；不直接 SQL；统一走 Domain / Platform Facade |

**为什么是五层而不是原图四层？**

| 原图 | 问题 | 五层对应 |
|---|---|---|
| 业务层 | 把「页面」和「领域规则」糊在一起 | 拆成 L2 Features + L3 Domain |
| 能力层 | 「业务无关」过强，登录/推送/图片掺业务语义 | 瘦身为 L4 Platform；语义升到 L3 |
| 基础设施层 | 组件库与崩溃混放尚可，但缺编排入口 | L5 Infra + 顶上补 L1 AppShell |
| 跨端层在最底 | 依赖方向倒置 | 移出依赖栈，作二期策略 |

#### 2.3.2 各层职责、输入输出、禁止事项

**L1 应用壳 — 编排与进程入口**

| 项 | 内容 |
|---|---|
| **存在理由** | 多业务共存需要唯一的启动、路由、门禁、全局 UI 宿主 |
| **拥有** | 路由表、Tab、深链分发、SessionGate、全局弹窗队列、初始化顺序 |
| **依赖** | L2（组装页面）、L3（问登录态/未读/强更）、L4/L5（启动期 SDK） |
| **不拥有** | Feed 分页、IM 消息序、token 刷新实现、上传算法 |
| **典型 API** | `Navigator.open(route)` / `SessionGate.ensureLogin()` / `DeepLink.handle(uri)` |

**L2 业务模块 — 产品功能的 UI 与用例编排**

| 项 | 内容 |
|---|---|
| **存在理由** | 按用户可感知功能垂直拆分，便于并行开发与独立迭代 |
| **拥有** | 页面、ViewModel、交互状态（输入中、列表滚动位置）、调用 Domain 的用例胶水 |
| **依赖** | 仅 L3 接口 + L5 UI 组件；经壳跳转其他 Feature |
| **不拥有** | 私有「全球单例网络」；不直接依赖其他 Feature 实现类 |
| **典型 API** | `FeedViewModel.refresh()` 内部调 `FeedSync.refresh()` |

一个 Feature 的标准形状：

```
feature-feed/
  ui/          # Page, Components（组合 L5 UIKit）
  vm/          # 订阅 Domain 流，转 UiState
  (可选) route # 向壳注册的路由定义
```

**L3 领域服务 — 业务规则、同步、跨页面复用的真相源**

| 项 | 内容 |
|---|---|
| **存在理由** | 社交的复杂度在「同步与规则」，不在控件；必须有稳定内核给多页复用 |
| **拥有** | 用例、领域模型、本地投影、同步引擎、乐观更新与回滚、幂等键策略 |
| **依赖** | L4（HTTP/WS/DB/PushChannel…）、L5（日志/DI） |
| **不拥有** | Activity/ViewController、路由、Toast 文案细节（可抛 DomainError 由 UI 映射） |
| **典型 API** | `FeedSync.timelineFlow` / `Conversation.sendText()` / `MediaPipeline.enqueue(draft)` |

L3 是**五层里最关键的业务资产**：换 UI 框架、甚至二期上 KMP，优先复用的是这一层的契约。

**L4 平台能力 — 可替换的技术能力 Facade**

| 项 | 内容 |
|---|---|
| **存在理由** | 把 OkHttp/APNs/Coil 等易变实现藏在稳定接口后，Domain 不绑死三方库 |
| **拥有** | `HttpClient`、`WsClient`、`KeyValue`、`SqlDriver`、`ImageLoader`、`PushChannel`、`Analytics`、`SecureStore` |
| **依赖** | 仅 L5 + 系统 SDK / 三方库 |
| **不拥有** | `Post`/`Message` 等业务实体；不 import feature-* / domain-* |
| **典型 API** | `http.get(path)` / `ws.connect()` / `db.execute()` / `push.register()` |

判断「该进 L3 还是 L4」：  
若去掉社交产品、换做工具 App 仍成立 → **L4**；若包含关注流/会话/媒资发布语义 → **L3**。

**L5 工程基建 — 开发期与运行期的通用底座**

| 项 | 内容 |
|---|---|
| **存在理由** | 所有层都要用的横切工具：UI 原子、DI、日志、崩溃、性能、构建 |
| **拥有** | Design System 原子组件、日志门面、Crash、Perf tracer、Module/DI 骨架 |
| **依赖** | 系统与三方；**禁止**依赖 L1–L4 业务 |
| **注意** | UI 组件库被 L1/L2 高频依赖，与 Crash 同属 L5，但对内可分子包 `:core:ui` / `:core:monitor` |

#### 2.3.3 依赖允许矩阵

行 = 谁发起依赖；列 = 被依赖方。`Y` = 允许，`N` = 禁止，`R` = 仅经路由/事件，不直接 import 实现。

|  | L1 壳 | L2 Feature | L3 Domain | L4 Platform | L5 Infra |
|---|---|---|---|---|---|
| **L1 壳** | — | Y（组装） | Y | Y（启动） | Y |
| **L2 Feature** | N（反依赖） | R（禁硬依赖） | Y | N（应经 L3） | Y（UI） |
| **L3 Domain** | N | N | Y（领域间接口） | Y | Y |
| **L4 Platform** | N | N | N | Y | Y |
| **L5 Infra** | N | N | N | N | Y |

补充规则：

1. **L2 → L4 默认禁止**：例如 UI 里直接 `api.like(postId)` 会绕过乐观更新与落库，导致多页状态分裂。例外仅限纯展示型极薄调用，且 0→1 阶段宁可禁止。  
2. **L2 ↔ L2**：只允许 `Navigator.open("im/chat?…")` 或订阅 L3 事件，禁止 `import ImChatActivity`。  
3. **L3 ↔ L3**：允许 `Create` 流程调 `MediaPipeline` + `FeedSync`；用接口，避免环依赖（必要时抽 `domain-contracts`）。

#### 2.3.4 一次用户操作如何穿过五层

**例：在 Feed 点赞**

```
L1  已在 main/feed，无路由变化
L2  FeedPage 点击 → FeedViewModel.onLike(postId)
L3  FeedSync.like(postId)
      → 乐观更新本地 DB 投影 → 通知 timelineFlow
      → 调 L4 HttpClient.post(/v1/posts/{id}/like, idempotency_key)
      → 失败则回滚投影并抛 DomainError
L4  附加 token、traceId、执行请求
L5  日志/埋点 SDK 被 L3 或 L4 调用 like_click / like_result
L2  收集 Flow 新状态 → UI 红心与计数变化
```

**例：推送点击进入聊天**

```
L4  PushChannel 回调 payload
L3  PushRouter 映射为 deeplink app://im/chat?session_id=
L1  DeepLinkCenter → SessionGate（未登录则先 auth/login 并记住回跳）
L1  Navigator.open(im/chat)
L2  ImChatPage 订阅 Conversation.messagesFlow(sessionId)
L3  Conversation 确保 WS/补洞后投影消息
```

**例：发布图文**

```
L1  打开 create/composer（可能经中心 Tab）
L2  CreatePage 编辑完成 → CreateViewModel.publish(draft)
L3  MediaPipeline 状态机：压缩(L4 ImageKit) → 直传(L4 Http) → 提交帖子
L3  成功后 FeedSync 插入/刷新
L2  展示进度；成功后 L1 可 navigate 回 Feed 或作者主页
```

#### 2.3.5 状态放在哪一层

| 状态类型 | 放置 | 例子 |
|---|---|---|
| 瞬时 UI | L2 | 输入框文案、滚动 offset、键盘是否弹出 |
| 业务真相 / 可复用 | L3 + 本地 DB | timeline 列表、会话未读、发送中消息 |
| 会话凭证 | L3 管理，L4 SecureStore 持久化 | access/refresh token |
| 设备/通道 | L4 | deviceId、push token 原始值 |
| 导航栈 | L1 | 当前 Tab、返回栈、待处理深链 |

原则：**同一业务事实只有一个写入口（一般在 L3）**；L2 只读投影 + 发意图。

#### 2.3.6 模块目录如何体现依赖

```
app/                    # L1 组装：Application、导航宿主、路由注册
feature/feed/           # L2
feature/im/
feature/create/
feature/profile/
feature/auth/
domain/feed/            # L3
domain/im/
domain/media/
domain/account/
domain/relation/
core/net/               # L4
core/storage/
core/push/
core/analytics/
core/ui/                # L5
core/monitor/           # L5 crash/perf/log
```

Gradle / SPM 依赖边与矩阵一致：`feature-feed` → `domain-feed` → `core-net`；**不存在** `core-net` → `feature-feed`。

可用 ArchUnit / 自定义 Detekt / Swift 禁令在 CI 卡反向依赖。

#### 2.3.7 常见错误对照

| 错误做法 | 后果 | 正确做法 |
|---|---|---|
| Feature 直接调 Raw HTTP | 多页状态不一致、难测 | 经 Domain |
| Domain 弹 Toast | 领域层绑 UI，难复用 | 抛错误码，L2/L1 呈现 |
| Platform 里写「关注流合并」 | 能力层被产品绑死 | 升到 FeedSync |
| 壳里写消息补洞 | 壳膨胀、IM 无法单测 | Conversation 领域 |
| Feature 互 import | 蝴蝶结依赖、无法拆包 | 壳路由 + Domain |
| 把跨端画在 L5 之下当地基 | 一期原生也被绑跨端叙事 | 跨端作二期策略 |

#### 2.3.8 和小团队怎么落地（避免过度设计）

0→1 不必先建 20 个 package，但**逻辑边界要先有**：

1. 物理上可先 `app` + `shared` 两模块，用**包名/文件夹**模拟五层。  
2. 出现第二个调用方（如主页也要帖子流）时，把逻辑**下沉 L3**，不要复制。  
3. 出现第二个技术实现（如换图片库）时，再抽 **L4 接口**。  
4. 始终先立规则：依赖方向 + 状态归属；目录可以后拆，规则不能后补才想。

### 2.4 仓库与工程形态

**推荐：双原生独立工程 + 共享契约仓库**

```
mobile-contracts/          # OpenAPI、错误码、埋点、深链、文案 key
ios-app/                   # Xcode workspace，SPM/模块化
android-app/               # Gradle multi-module
design-tokens/             # 颜色、字号、间距（导出双端）
```

| 方案 | 何时选 |
|---|---|
| 双原生独立工程（默认） | 0→1、团队有双端人力、要控体验与性能 |
| 单端先发 + 另一端跟随 | 人力极度不足时；契约仍要先写 |
| RN/Flutter 主导 | 仅当明确「业务页同构」优先于原生体验，且团队栈匹配 |

一期默认：**Kotlin + Swift（或 SwiftUI 渐进）双原生**。

---

## 3. 模块地图

### 3.1 L1 应用壳

#### 为什么在「最上面」

图里越靠上 = **越靠近用户、依赖越多**，不是「越重要」或「越底层」。

| 误解 | 正确理解 |
|---|---|
| 壳是地基，所以应画在最下 | 地基是 L4/L5（网络、存储、崩溃）；壳是**组装与编排** |
| 壳最上面 = 壳最核心 | 核心业务规则在 L3；壳只决定「怎么进页、谁先初始化」 |
| 业务模块可以自己管路由/登录拦截 | 那样深链、推送、未登录回跳会在各 Feature 复制且互相打架 |

应用壳在顶层，是因为：

1. **它依赖几乎所有下层**（要组装 Feature、调 Domain、用 Platform），按依赖方向只能在最上。
2. **它不承载领域规则**（不写 Feed 去重、不写 IM seq），只做生命周期与导航编排。
3. **跨业务跳转的唯一枢纽**：Feed → 个人主页 → 私信，不允许 Feature 互相硬依赖，必须经壳的路由表。
4. **进程级单例职责**：启动顺序、全局弹窗、会话门禁、深链，天然属于 App 进程入口，不属于某个业务包。

```
用户可见入口
    ↓
L1 应用壳（编排：启动 / 路由 / 门禁 / 全局 UI）
    ↓ 组装
L2 各业务页面（只关心自己的 UI 与用例）
    ↓
L3 领域（真正的业务规则与同步）
```

#### 壳里要设计什么（清单）

| 子系统 | 要设计的内容 | 一期必须？ |
|---|---|---|
| **启动与初始化** | 冷/热启动路径；SDK 初始化顺序与超时；隐私合规同意后再采数；闪屏/首屏策略 | 是 |
| **信息架构 / Tab** | Tab 结构、中心创作按钮、未读红点归属（消息 Tab） | 是 |
| **路由表** | 页面名 ↔ 组件；参数校验；未知路由兜底页 | 是 |
| **导航栈策略** | Tab 内 Stack、全局 Modal、透明登录页；返回栈清理规则 | 是 |
| **深链** | `app://` / Universal Link / App Link schema；版本兼容；冷启动深链延迟到壳就绪 | 是 |
| **推送点击入口** | payload → 深链；与前台横幅去重 | 是 |
| **SessionGate** | 哪些路由要登录；登录成功回跳；踢下线全站处理 | 是 |
| **全局 UI 宿主** | Toast / Loading / 对话框队列；强制更新；公告/停服 | 是 |
| **账号切换 / 登出** | 清栈回登录；清敏感缓存的触发点（调 Domain，不自己删库细节） | 是 |
| **配置门面** | 远程配置拉取失败兜底；功能开关关闭时隐藏 Tab/入口 | 是 |
| **生命周期** | 前后台切换时通知 IM 长连接降频/断开、埋点 `foreground` | 是 |
| **权限编排** | 通知/相册等「何时申请」的统一策略（避免业务页乱弹） | 建议 |
| **多 Window / 分屏**（后置） | 平板、折叠屏导航适配 | 否 |
| **插件/RN 容器宿主**（二期） | 若引入跨端页，壳负责容器与原生路由互跳 | 否 |

#### 组件职责表

| 组件 | 职责 | 不该做的事 |
|---|---|---|
| `AppLaunch` | 冷/热启动、隐私弹窗、SDK 初始化顺序 | 请求 Feed 业务数据细节 |
| `Navigator` | Tab + Stack + 模态；统一 `navigate(route, args)` | 拼装业务 API 参数逻辑 |
| `DeepLinkCenter` | URL / 推送 click → 标准 Route | 解析后直接 new 业务 VC 绕过路由表 |
| `SessionGate` | 未登录拦截、登录后回跳、踢下线 | 实现登录协议 / 存 token（属 Domain/Platform） |
| `GlobalUI` | Toast、Loading、强制更新、公告队列 | 业务表单校验 |
| `BadgeHub`（可挂壳） | 汇总 IM 未读等到 Tab 角标 | 自己维护消息序（属 Conversation） |

#### 启动顺序（建议）

1. 本地配置 / 安全存储  
2. 日志 & 崩溃  
3. 远程配置（短超时 + 本地兜底）  
4. 账号会话恢复（AccountSession）  
5. 网络与推送注册  
6. 挂载壳（Tab）→ 按需触发 Feed/IM 同步  
7. 若有待处理深链 / 推送 click → 再导航  

#### 路由表示例（一期）

| Route | 登录？ | 目标 |
|---|---|---|
| `main/feed` | 可选或要求（产品定） | Feed Tab |
| `main/im` | 是 | 消息 Tab |
| `main/profile` | 是 | 我的 |
| `post/detail?id=` | 否/是 | 帖子详情 |
| `profile/user?id=` | 否 | 他人主页 |
| `im/chat?session_id=` | 是 | 聊天页 |
| `create/composer` | 是 | 创作 |
| `auth/login` | 否 | 登录 |

**跨 Feature 跳转约定：** 只允许 `Navigator.open(route)`，禁止 `Feed` import `IMChatPage`。

#### 壳与下层的边界（一句话）

壳问 Domain：「当前是否已登录 / 未读多少 / 是否强制更新」；壳不实现「如何同步未读 / 如何刷新 token」。

### 3.2 L2 业务模块

| Feature | 页面/能力 | 依赖 Domain |
|---|---|---|
| `feature-auth` | 登录、绑定、资料编辑入口 | AccountSession |
| `feature-feed` | 信息流、详情、评论列表 | FeedSync, Relation, Media |
| `feature-profile` | 主页、关注列表、粉丝 | Relation, FeedSync |
| `feature-create` | 相机/相册、编辑、发布进度 | MediaPipeline, FeedSync |
| `feature-im` | 会话列表、聊天页 | Conversation, PushRouter |
| `feature-ai`（可选） | AI 会话 UI | 复用 Conversation 壳或独立 LLM Gateway |

### 3.3 L3 领域服务（核心）

#### AccountSession
- 登录、刷新 token、登出、踢下线
- 当前用户资料缓存
- 登录态变更事件总线

#### Relation
- 关注/取关、关系状态、粉丝/关注分页
- 主页聚合（资料 + 计数 + 关系）

#### FeedSync
- Timeline 拉取（cursor）、本地缓存、去重
- 互动：赞、评、删评（乐观更新 + 失败回滚）
- 与 Media 协作展示封面/原图 URL

#### Conversation（IM）
- 会话列表、消息收发、本地消息序、发送状态机
- 未读计数、会话草稿
- 通过 WS/长连接收推，HTTP 补历史与补洞

#### MediaPipeline
- 选图 → 压缩/转码 → 直传凭证 → 分片/整传 → 提交 contentId
- 发布任务队列（可后台续传，受系统限制）
- 失败重试、进度回调

#### PushRouter
- 将推送 payload 映射为 DeepLink / 未读增量
- 与 Conversation、Feed 的未读模型对齐

#### ConfigGate
- 功能开关、灰度、强制更新、文案/CDN 域名热更新

### 3.4 L4 平台能力

| 能力 | 一期选型建议 | 说明 |
|---|---|---|
| HTTP | OkHttp / URLSession + 统一拦截器 | 鉴权、重试、签名、traceId |
| WS/长连接 | OkHttp WS / Starscream 或自研轻量 | IM、在线状态；断线指数退避 |
| 存储 | DataStore/MMKV + SQLite/Room；iOS UserDefaults/Keychain + GRDB/CoreData/SQLite | KV 与结构化分离 |
| 图片 | Coil / SDWebImage 或 Kingfisher | 解码缓存；业务裁剪在 Domain |
| 推送 | FCM + 厂商通道；APNs | 通道在 L4，路由在 L3 |
| 埋点 | 自研轻量 + 可接三方 | 先定事件字典 |
| 安全存储 | Keychain / EncryptedSharedPreferences | token 永不落明文日志 |

### 3.5 L5 基建

- **UI 组件库**：按钮、列表、头像、图片浏览器、空态/错态；双端视觉对齐
- **DI / 模块化**：Android Hilt/Koin + Gradle modules；iOS SPM + 协议注入
- **日志**：分级、隐私脱敏、可上传采样
- **崩溃 / ANR**：Firebase Crashlytics 或 Bugly 等
- **性能**：启动耗时、Feed 首帧、滚动掉帧、包体积基线
- **CI**：lint、单测、截图/UI 烟测、打包符号表上传

---

## 4. 核心链路设计

### 4.1 账号与会话

```
登录成功 → 写 SecureStore(token, refresh)
        → AccountSession.emit(LoggedIn)
        → 注册推送 deviceToken
        → 触发 FeedSync.warm() + Conversation.syncSessions()
```

- Access Token 短有效；Refresh 走单飞（single-flight）避免并发刷新打爆
- 401 统一拦截：刷新失败 → 清会话 → 跳登录并保留深链回跳
- 多端互踢：服务端推 `session_revoked`，客户端强制登出

### 4.2 Feed 消费

```
进入 Feed Tab
  → Domain.FeedSync.observeTimeline()  // UI 只订阅
  → 若本地有缓存先渲染
  → refresh(cursor=null) / loadMore(cursor)
  → 合并去重写入本地 → 通知 UI
```

**协议约定（示例）**

- `GET /v1/feed/timeline?cursor=&limit=`
- 返回 `items[]` + `next_cursor` + `server_time`
- 客户端以 `post_id` 去重；同 id 以 `updated_at` 较大者覆盖

**互动**

- 赞：乐观 +1，请求失败回滚并 toast
- 评论：先发 pending，成功替换真实 id

### 4.3 创作发布

```
编辑完成
  → MediaPipeline.createTask(draft)
  → 申请 upload token
  → 上传图片(可并行受限并发)
  → POST /v1/posts { media_ids, text, ... }
  → 本地插入「发布中/已发布」占位
  → 成功后刷新作者主页 & 可选插入关注流
```

**状态机**：`Draft → Preparing → Uploading → Submitting → Success | Failed(可重试)`

失败策略：网络错误自动重试有限次；业务错误（违规）不可自动重试，展示原因。

### 4.4 IM 私信

```
发送：
  生成 client_msg_id
  → 本地插入 status=sending
  → WS send 或 HTTP fallback
  → ack 后 status=sent，写入 server_msg_id/seq

接收：
  WS push → 校验 seq
  → 连续则写入；缺口则 HTTP 补洞
  → 更新会话 last_message / unread
```

**一期范围**

- 文本 + 图片消息
- 单聊
- 会话列表排序（时间）
- 推送拉起聊天页

**后置**：群聊、已读、撤回、语音、通话。

### 4.5 推送与深链

Payload 最小约定：

```json
{
  "type": "im.message",
  "deeplink": "app://im/chat?session_id=...",
  "badge": 3,
  "extras": {}
}
```

- 点击：DeepLinkCenter 解析 → SessionGate → 目标页
- 前台：IM 类走应用内横幅；勿与系统通知重复骚扰（可配置）

---

## 5. 数据与同步架构

### 5.1 存储分区

| 区域 | 内容 | 生命周期 |
|---|---|---|
| Secure | token、refresh、设备密钥 | 登出清除 |
| User KV | 偏好、草稿索引、开关缓存 | 可随用户 |
| Feed DB | timeline、post 详情、评论缓存 | 可 LRU 淘汰 |
| IM DB | sessions、messages、未读 | 登出可清或保留加密 |
| Media | 临时压缩文件、上传任务 | 成功后清理 |

### 5.2 同步模型

- **Feed**：拉取式 + 本地缓存；不强求实时全量推送（点赞数可进页刷新）
- **IM**：推送式（WS）+ 拉取补洞；以 `session_id + seq` 为序
- **关系**：写后读；列表分页；本地可缓存关系状态 map

### 5.3 一致性原则

1. UI 永不直接写网络响应到视图模型长期态；统一经 Domain 落库再投影  
2. 乐观更新必须可回滚  
3. 所有写操作带幂等键（`client_request_id` / `client_msg_id`）  
4. 时钟以 `server_time` 校准展示「刚刚」等相对时间  

---

## 6. 网络与实时

### 6.1 HTTP

- Base URL 可远程配置；证书锁定可二期
- 统一 `RequestId` / `traceparent` 便于排障
- 重试：仅幂等 GET 与明确可重试写；POST 发布/发消息靠幂等键而非盲目重试

### 6.2 长连接

- 连接态：`Idle → Connecting → Auth → Ready → Backoff`
- 心跳；后台降频或断开（遵守 iOS/Android 后台策略）
- IM 发送：Ready 走 WS，否则 HTTP；避免双通道重复需服务端去重

---

## 7. 媒资方案

### 7.1 图片一期

1. 系统相册/相机选取  
2. 按场景压缩（Feed 长边上限、JPEG/WebP 质量档）  
3. 直传对象存储（预签名 URL）  
4. 服务端返回 `media_id` + CDN URL 模板  
5. 展示侧用宽度描述符（或多码率 URL）  

### 7.2 视频二期前置预留

- Domain 的 `MediaPipeline` 预留 `VideoTranscode` 接口  
- 一期 UI 可隐藏入口，避免半成品视频链路拖垮发布成功率  

---

## 8. 可观测性与质量

### 8.1 核心指标（客户端）

| 指标 | 目标参考（上线初期） |
|---|---|
| 冷启动至首屏可交互 | 视机型定基线，持续下降 |
| Feed 首屏成功率 | > 99%（含缓存命中） |
| 图文发布成功率 | > 98% |
| IM 消息发送成功率 | > 99% |
| 崩溃率 | < 0.1% 会话或团队约定 |
| ANR / 卡死 | 纳入周报 |

### 8.2 埋点字典（最小集）

- `app_launch` / `app_foreground`
- `feed_expose` / `feed_refresh` / `post_click` / `like_click` / `comment_submit`
- `create_start` / `create_upload_success` / `create_publish_result`
- `im_session_open` / `im_msg_send_result`
- `push_recv` / `push_click`
- `login_result` / `logout`

公共字段：`user_id`, `device`, `app_ver`, `os`, `network`, `trace_id`。

### 8.3 日志与隐私

- 默认脱敏：token、手机号、精确定位  
- 日志上传采样；IM 内容默认不上报明文  

---

## 9. 安全与合规

1. 传输 HTTPS；敏感字段可额外加密（按合规要求）  
2. Token 仅存安全区；禁止写入 WebView Cookie 明文长期态（若有 H5）  
3. 权限最小化：相册、相机、通知按需申请，有拒绝后引导  
4. 隐私政策 / 权限说明在初始化采集前展示（按地区法规）  
5. 截屏敏感页可二期处理；一期至少聊天页考虑 FLAG_SECURE（可配置）  

---

## 10. 技术选型清单（一期推荐）

| 域 | Android | iOS |
|---|---|---|
| 语言 | Kotlin | Swift |
| UI | Jetpack Compose（或 XML 渐进） | UIKit 为主，SwiftUI 局部渐进 |
| 异步 | Coroutines + Flow | async/await + Combine/AsyncStream |
| 架构模式 | 每 Feature：UI → ViewModel → Domain UseCase | 同左 |
| 本地 DB | Room (SQLite) | GRDB 或 SQLite.swift |
| 图片 | Coil | Kingfisher |
| DI | Hilt | 工厂 + 协议，或 Needle/Swinject（保持轻） |
| 推送 | FCM + 厂商 | APNs |
| 崩溃 | Crashlytics / 等价 | 同左 |
| 包管理 | Gradle Version Catalog | SPM |

**明确不选（一期）**：在客户端同时引入 RN 容器 + KMP 共享库作为主路径。

---

## 11. 模块化落地（示例）

### Android Gradle 模块

```
:app
:core:ui
:core:net
:core:storage
:core:analytics
:domain:account
:domain:feed
:domain:im
:domain:media
:domain:relation
:feature:feed
:feature:profile
:feature:create
:feature:im
:feature:auth
```

### iOS SPM 产品

与上表同名 Target；`App` 只组装，不写业务细节。

### 接口隔离示例

```text
FeedRepository (protocol)
  refresh()
  loadMore()
  timelineFlow() -> Stream<List<Post>>
  like(postId)
```

`feature-feed` 只依赖协议；实现与 Room/网络在 `domain-feed`。

---

## 12. 与服务端的契约

双端共用 `mobile-contracts`：

1. **OpenAPI 3** 描述 REST  
2. **异步事件**（WS）：`message.new` / `message.ack` / `session.updated` / `session.revoked`  
3. **错误码**：`AUTH_EXPIRED` / `RATE_LIMITED` / `CONTENT_VIOLATION` …  
4. **深链 schema** 文档化并做兼容测试  
5. **分页**：统一 cursor，禁止双端各用不兼容 page/offset  

变更流程：契约 PR → 生成/校验客户端 → 再合业务。

---

## 13. 分期路线图

### Phase 0 — 地基（并行于产品原型）

- 工程骨架、DI、路由、网络、登录会话、崩溃与埋点脚手架  
- 契约仓库与 CI  
- 设计 tokens / 基础组件  

### Phase 1 — 社交闭环 MVP

- 关系链 + 主页  
- Feed 读与赞评  
- 图文发布  
- 私信单聊 + 推送  
- 远程配置与强制更新  

**验收：** 新用户 10 分钟内完成注册→关注→发帖→互动→私信。

### Phase 2 — 同步与体验加固

- Feed/IM 离线缓存与补洞完善  
- 发布任务可靠性、弱网优化  
- 性能基线与包体积治理  
- 评论体验、分享回流  

### Phase 3 — 增长与可选跨端

- 短视频（若产品需要）  
- 搜索 / 话题 / 简易推荐  
- AI 对话（独立开关）  
- **跨端二选一试点**：  
  - 选 RN：先迁 1–2 个运营向页面  
  - 选 KMP：先迁 Account/Feed 领域模型与校验逻辑  
  - SwiftUI：仅 iOS 新页面渐进，不定义跨端战略  

---

## 14. 团队分工建议（小团队）

| 角色 | 负责 |
|---|---|
| 客户端 Owner | 分层规则、契约、质量指标 |
| Android / iOS Dev | Feature 垂直交付，复用 Domain 接口 |
| 后端 | Timeline、关系、媒资、IM、推送 |
| 设计 | 双端组件与关键路径一致性 |
| 测试 | 闭环用例、弱网、推送、升级、权限拒绝路径 |

初期避免「纯平台组」过大；平台能力从业务抽取，而不是先造大中台。

---

## 15. 风险与对策

| 风险 | 对策 |
|---|---|
| IM 做成普通 CRUD | 一开始就上 client_msg_id + seq 补洞 |
| 发布成功率低 | 状态机 + 可重试任务 + 指标看板 |
| 双端行为不一致 | 契约测试 + 埋点对齐评审 |
| 过早跨端 | 写入非目标；Phase 3 再评估 ROI |
| 能力层塞业务 | Code review 卡依赖方向 |
| 启动堆太多 SDK | 初始化分阶段；非关键懒加载 |

---

## 16. 决策记录（ADR 摘要）

| ID | 决策 | 状态 |
|---|---|---|
| ADR-001 | 一期双端原生，不做跨端主路径 | 采纳 |
| ADR-002 | 五层依赖 + 跨端作策略而非底层 | 采纳 |
| ADR-003 | Feed/IM 为带私有存储的领域垂直切片 | 采纳 |
| ADR-004 | UI 状态经 Domain 落库再投影 | 采纳 |
| ADR-005 | 二期跨端 RN 与 KMP 择一 | 待 Phase 2 末评估 |
| ADR-006 | AI 对话非 MVP 必做 | 采纳 |

---

## 17. 交付物清单

- [x] 分层评审文档  
- [x] 本完整设计方案  
- [ ] `mobile-contracts` 初版（OpenAPI + 错误码 + 埋点字典）  
- [ ] iOS/Android 工程骨架与模块边界  
- [ ] 闭环自动化冒烟用例列表  
- [ ] 质量指标看板定义  

---

## 附录 A. 信息架构（App 导航）

```
Tab
├── 首页 Feed
├── 发现（二期可开；一期可隐藏或做简单关注推荐）
├── 创作（中心按钮 → Create）
├── 消息 IM
└── 我的 Profile
```

## 附录 B. 目录级依赖示意

```
feature-*  →  domain-*  →  core-*
app        →  feature-* + domain-* + core-*
core-*     ↛ feature-* / domain-*
```

## 附录 C. 相关文档

- [分层评审](./social-app-mobile-architecture.md)  
- [架构摘要](../docs/architecture/social-mobile-0-to-1.md)  
