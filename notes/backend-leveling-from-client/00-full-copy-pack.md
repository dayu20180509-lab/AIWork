# 客户端转后端：练级 + 详解 + 面试题（完整可复制版）

> 由 `01-leveling-guide.md` 及全部专题文档展开合并，便于一次性复制/导出。
> 含：主攻略、L0 HTTP/Node 面试详解、最小栈、L1～L5、日志冷热、面试题（含 I5 Node）。
> 原拆分文件仍可单独阅读。参考专栏：《左耳听风》https://time.geekbang.org/column/intro/100002201

---

# 目录

1. [练级主攻略（01）](#part-01)
2. [L0 HTTP 服务端视角 + Node 落地（11）](#part-11)
3. [L0 Node.js 面试向详解（12）](#part-12)
4. [L0 最小后端栈（03）](#part-03)
5. [L1 入门与修养（04）](#part-04)
6. [L1 关系数据库基础（05）](#part-05)
7. [L2 专业基础（06）](#part-06)
8. [L3 软件设计与范式（07）](#part-07)
9. [L4 分布式弹力与性能（08）](#part-08)
10. [L5 架构进阶与高手成长（09）](#part-09)
11. [日志冷热 HDFS + ES + Nginx（10）](#part-10)
12. [面试题与参考答案（02）](#part-02)

---

<a id="part-01"></a>

# ========== PART 01：练级主攻略（01） ==========

# 客户端背景：后端开发练级攻略

> 结构对齐《左耳听风》「程序员练级攻略」五篇（入门 / 修养 / 专业基础 / 软件设计 / 高手成长），并嵌入「分布式」「弹力设计」「性能设计」「管理设计」等后端核心模块。  
> 目标读者：**有客户端工程经验，要系统转向后端 / 全栈 / 服务端架构**。  
> 需要一次性复制全文时，见展开合集：[00-full-copy-pack.md](./00-full-copy-pack.md)（含本文 + 文内全部链接详解 + 面试题）。

---

## 0. 开篇：为什么客户端转后端是「降维」而不是「从零」

客户端工程师通常已经具备：

| 客户端已有能力 | 后端对应能力 | 迁移要点 |
|----------------|--------------|----------|
| HTTP/HTTPS、REST、Token、证书钉扎 | API 设计、鉴权、TLS 终结 | 从「消费方」变成「提供方」 |
| 本地 DB（SQLite/Realm）、缓存 | MySQL/PG、Redis | 从单机持久化 → 共享、事务、并发写 |
| GCD / 协程 / Rx / 线程池 | 服务并发、连接池、异步 IO | 关注点从 UI 流畅度 → 吞吐与稳定性 |
| 状态机、页面缓存、乐观更新 | 幂等、最终一致性、补偿 | UI 乐观更新 ≈ 后端最终一致 |
| Crash、ANR、埋点、灰度 | 监控、熔断、限流、降级、发布 | 线上故障处理同构，工具不同 |
| 包体积、启动优化、电量 | 延迟、QPS、容量规划 | 资源约束从设备 → 集群 |

《左耳听风》反复强调：**不要追名词，要追本质；不要只堆业务，要补理论与系统知识**。转后端同样如此——语言可以学 Go/Java/Kotlin，但真正拉开差距的是：**数据、并发、故障、一致性与工程规范**。

### 学习态度（对齐专栏建议）

1. **坚持**：后端知识面宽，按级别推进，不要三天换一门框架。
2. **动手**：每个概念至少做一个可运行的小服务。
3. **思考**：每个「为什么这样设计」都要能对照客户端场景回答。
4. **重基础**：网络、OS、数据结构、数据库原理比换框架更保值。
5. **取法其上**：标准可以定高一点；达不到也能显著高于「只会 CRUD」。

---

## 1. 心智模型：客户端视角 vs 服务端视角

```
客户端心智：
  用户操作 → 本地状态 → 调 API → 渲染 / 缓存 / 重试

服务端心智：
  流量入口 → 鉴权/限流 → 业务逻辑 → DB/缓存/MQ → 可观测性 → 故障与弹力
```

### 五个必须建立的后端心智

1. **共享可变状态是默认敌人**  
   客户端多为单用户本地状态；后端是多租户、多实例共享数据。凡写操作先想：并发、幂等、事务边界。

2. **失败是常态**  
   网络超时、下游挂掉、磁盘慢、发布误操作都会发生。专栏「弹力设计」整章就是在讲：**隔离、异步、幂等、重试、熔断、限流、降级、补偿**。

3. **一致性是产品决策，不是纯技术选择题**  
   强一致贵；最终一致要配补偿与可观测。下单、支付、库存、消息已读，策略不同。

4. **接口是契约，版本是负债**  
   你曾痛恨后端乱改字段；现在你要成为「不乱改字段」的人：兼容、废弃、文档、契约测试。

5. **可运维性 = 功能的一部分**  
   没有日志/指标/链路追踪的功能，等于上线即黑盒。

---

## 2. 练级总览（L0 → L5）

| 级别 | 角色画像 | 对齐《左耳听风》 | 出师标准（可自检） |
|------|----------|------------------|--------------------|
| L0 转型桥梁 | 会调 API 的客户端 | 入门篇 + 网络基础 | 独立写一个带鉴权的 CRUD API |
| L1 后端入门 | 能交付业务接口 | 正式入门 + 修养篇 | 熟悉一种后端语言栈 + Git/CI + SQL |
| L2 专业基础 | 懂系统与数据 | 专业基础篇 + 数据库 | 能解释索引、事务、TCP、进程模型 |
| L3 工程设计 | 能设计模块与 API | 软件设计 + 编程范式 | 能做领域拆分、SOLID、错误处理规范 |
| L4 分布式与弹力 | 能扛故障与规模 | 分布式 + 弹力/性能/管理设计 | 能讲缓存、MQ、熔断限流、部署策略 |
| L5 架构进阶 | 能选型与取舍 | 高手成长篇 | 能做容量评估、SLA、技术债与演进路线 |

---

## 3. L0 — 转型桥梁（2–4 周）

### 目标

把「我是 API 调用方」变成「我是 API 提供方」。

### 必学

1. **HTTP 再学一遍（服务端视角）**（详解 + Node 落地见 [11-http-server-nodejs.md](./11-http-server-nodejs.md)）  
   - 方法语义：GET 安全幂等；PUT/DELETE 幂等；POST 不幂等（需设计幂等键）  
   - 状态码分层：2xx 成功 / 4xx 客户端错 / 5xx 服务端错  
   - Header：`Authorization`、`Idempotency-Key`、`ETag`/`If-None-Match`、分页与限流头  
   - 客户端经验映射：你写的重试逻辑，后端必须配合幂等与正确状态码  
   - Node：Method 路由、鉴权/幂等中间件、统一错误、`Retry-After`、与日志项目 GET 回填的关系

2. **选一门后端主语言（建议三选一）**  
   - **Go**：部署简单、并发模型清晰，适合服务端入门（专栏对 Go/Docker 有专门讨论）  
   - **Java / Kotlin**：生态全（Spring），面试与大厂后端岗位覆盖广  
   - **Node / TypeScript**：若你来自 RN/前端，迁移成本最低，但要补强类型与并发边界（**面试向详解**：[12-l0-nodejs-interview.md](./12-l0-nodejs-interview.md)）  

   《练级攻略》偏 Java 作为工业级主语言；**客户端转后端可优先 Go 或 Kotlin**；若选 Node，按 12 补齐事件循环、中间件、连接池与幂等话术。

3. **最小后端栈**（详解见 [03-minimal-backend-stack.md](./03-minimal-backend-stack.md)）  
   - **Web 框架 + 路由**：Method+Path → Handler；中间件链 ≈ 客户端拦截器；统一 JSON 错误  
   - **配置（环境变量）**：同一镜像多环境；密钥不进 git；必填项缺失则启动失败  
   - **结构化日志**：一行一条 JSON；必带 `request_id` / method / path / status / latency  
   - **健康检查 `GET /healthz`**：给负载均衡/K8s 探活；L1+ 再拆 liveness / readiness  
   - **Docker 一键跑起来**：多阶段构建 + Compose；新机器只装 Docker 也能通探活  

   验收口令：任意装了 Docker 的机器上，只改环境变量、不改代码，服务能起来并证明活着。

### 实践项目 A：个人书签 / 待办 API

- 注册登录（JWT 或 Session）  
- CRUD + 分页 + 过滤  
- 请求 ID 贯穿日志  
- OpenAPI / Swagger 文档  

**验收**：用你熟悉的客户端（iOS/Android）对接自己的 API，并故意做：重复提交、弱网重试、Token 过期——观察服务端行为是否正确。

### 专栏对照

- 011/012 错误处理（返回码 vs 异常、异步错误）  
- 069–071 练级攻略入门篇  
- 091–094 面试攻略（可并行浏览态度与准备方法）

---

## 4. L1 — 后端入门与修养（4–8 周）

> 详解：[04-l1-entry-and-craft.md](./04-l1-entry-and-craft.md)（修养操练、工具链、SQL、项目 B 排期与验收清单）

前置：L0 最小栈已通。L1 要把「能跑 Demo」做成「能上预发」——真库、真鉴权、真协作规范。

### 4.1 程序员修养（转后端尤其重要）

客户端常「本地试、提测、发版」；后端是「共享环境、数据不可回滚、故障影响面大」。重点练：

| 修养项 | 具体要求 |
|--------|----------|
| 提问与排查 | 先日志/指标/复现步骤，再问人；会写最小复现 |
| 安全意识 | SQL 注入、越权、敏感日志脱敏、密钥不进仓库 |
| 上线规范 | 迁移脚本可回滚、特性开关、灰度、回滚预案 |
| 代码规范 | 统一错误码、分层（handler/service/repo）、禁止业务逻辑散落 |
| 英文与文档 | 会读官方文档与 RFC；接口文档当作产品交付 |

### 4.2 工程工具链

- Git 工作流（Feature Branch / PR Review）——专栏有 Git 协同工作流专章  
- 单测 + 集成测（至少覆盖鉴权与核心写路径）  
- CI：lint、test、镜像构建  
- 本地依赖：Docker Compose 起 MySQL/Postgres（+ Redis 可占位）  
- 连接池总账：实例数 × 池大小 ≤ DB 最大连接 × 安全系数  

### 4.3 关系数据库入门

> 详解：[05-relational-db-basics.md](./05-relational-db-basics.md)

- 表设计：主键、唯一约束、外键策略、软删除  
- 索引：B+Tree 直觉、最左前缀、覆盖索引；列表接口上线前跑 `EXPLAIN`  
- 事务：ACID、隔离级别；写路径显式事务，事务内禁止调外部 HTTP  
- 迁移工具：Flyway / Liquibase / golang-migrate  

### 实践项目 B：多用户笔记服务

- 用户 / 笔记本 / 笔记三表  
- 列表接口必须有索引支撑（用 `EXPLAIN` 验证）  
- 写操作带事务；唯一约束防重复标题  
- 简单 RBAC：只能改自己的笔记（换 Token 做越权测试）  

**验收**：压测工具（k6 / wrk）打列表接口，看慢查询与连接池表现；CI 绿；迁移有回滚说明。

**出师口令**：别人能靠 Compose 跑起你的服务；你能证明列表走索引、写路径安全且可回滚。

### 专栏对照

- 072 程序员修养  
- 080 数据库  
- 031 Git 协同工作流

---

## 5. L2 — 专业基础（持续 2–3 个月，与工作并行）

> 详解：[06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md)（网络与 Linux、数据结构够用版、缓存、项目 C 排期）

《专业基础篇》三块：**编程语言深度、理论学科、系统知识**。客户端转后端建议优先级如下。

### 5.1 系统与网络（必补）

客户端已接触 Socket/HTTP，但后端需要：

- TCP 握手、滑动窗口、TIME_WAIT、连接复用；**超时预算层层缩短**  
- HTTP/1.1 Keep-Alive、HTTP/2 多路复用、反向代理角色  
- 进程 / 线程 / 协程；阻塞 IO vs 非阻塞 / 异步 IO（专栏 077/078）  
- Linux 基础：进程、文件描述符、`ulimit`、`ss`/`top`/`lsof`  

**客户端映射**：OkHttp 连接池 ≈ 服务端 Upstream Keep-Alive；ANR ≈ 服务端线程池打满。

### 5.2 数据结构与算法（够用即可，定向加深）

- 哈希、跳表、堆、树：理解 Redis / 索引 / 优先队列为何快  
- 复杂度直觉：避免在请求路径 `O(n²)` 扫表与 N+1 查询  
- 面试向：LeetCode Medium 为主；映射到限流、LRU、调度  

### 5.3 缓存

专栏性能设计「缓存」是后端分水岭：

- 本地缓存 vs 分布式缓存  
- 缓存穿透 / 击穿 / 雪崩  
- 旁路缓存（Cache-Aside）最常用；写后**删**缓存  
- TTL 加随机抖动；用 hit/miss 日志验证一致性窗口  

**客户端映射**：Memory + Disk 二级缓存、图片缓存淘汰 ≈ LRU/LFU。

### 实践项目 C：Feed 流或商品列表

- DB + Redis 缓存热点列表  
- 缓存 miss 回源；设置随机 TTL 防雪崩  
- 写后删缓存；压测对比开关缓存的 P95 / DB QPS  

**出师口令**：能说明慢在连接/池/SQL/缓存哪一层，并拿出加缓存前后的数字。

### 专栏对照

- 074 理论学科 / 075 系统知识  
- 077 Linux、内存和网络 / 078 异步 IO 与 Lock-Free  
- 064 性能设计：缓存

---

## 6. L3 — 软件设计与编程范式（贯穿始终）

> 详解：[07-l3-design-and-paradigms.md](./07-l3-design-and-paradigms.md)（原则、范式、错误模型、项目 D 重构步骤）

后端接口一多，没有设计就会变成「上帝 Service」。L3 **贯穿始终**：与 L2/L4 并行，用边界约束正在写的代码。

### 6.1 设计原则（与客户端同构）

- DRY / KISS / YAGNI  
- SOLID  
- 迪米特法则（最少知识）  
- 关注点分离：协议层、领域层、基础设施层（Handler → Service → Repo）

### 6.2 编程范式（专栏「编程范式游记」）

不必全学，但要能在后端落地：

| 范式 | 后端用法 |
|------|----------|
| OOP | 领域模型、聚合根边界 |
| 函数式 | 纯函数核心逻辑、易测 |
| 泛型 | Repository / 中间件抽象（有证据再抽） |
| 装饰器 / 中间件 | 鉴权、日志、限流链式处理 |

### 6.3 错误处理

对齐专栏 011/012：

- 业务错误 vs 系统错误要分层  
- 不要把异常当控制流到处抛  
- 异步与消息消费的错误要可重试、可死信  
- 对外错误码稳定，对内日志带上下文  

### 实践项目 D：把项目 B/C 重构为清晰分层

- `handler` 只做协议转换  
- `service` 做业务规则与事务边界  
- `repo` 做持久化  
- 引入领域错误类型；单测覆盖核心规则（不启 HTTP 也能测）  

**出师口令**：新功能默认知道放哪一层；错误码稳定；横切走中间件。

### 专栏对照

- 028–040 编程范式游记  
- 076 软件设计  
- 011/012 错误处理

---

## 7. L4 — 分布式、弹力与性能（后端核心战场）

> 详解：[08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md)（难点、弹力组合、管理/性能、项目 E 演练）

这是客户端转后端的「主副本」。专栏把分布式拆成：**难点认知 → 技术栈 → 监控/调度 → 弹力 → 管理 → 性能**。

### 7.1 分布式难点（先建立问题意识）

亚马逊实践与分布式「冰与火」告诉我们常见难点：

- 部分失败、时钟不可靠、网络分区  
- 状态难共享、部署与回滚复杂  
- 故障诊断需要全链路视角  

**客户端映射**：弱网重试导致「下单两次」——本质就是分布式部分失败 + 缺幂等。

### 7.2 技术栈心智图

```
接入层：DNS / CDN / 网关 / 负载均衡
服务层：无状态服务 + 服务发现 + 配置中心
数据层：主从 / 分片 / 缓存 / 对象存储
异步层：消息队列 / 任务调度 / 工作流
观测层：日志 / 指标 / 链路追踪 / 告警
运维层：容器编排 / 发布策略 / 弹性伸缩
```

### 7.3 弹力设计清单（必须会讲、会用）

| 模式 | 一句话 | 客户端类比 |
|------|--------|------------|
| 隔离（Bulkhead） | 故障隔舱，避免拖垮全部 | 图片线程池与网络线程池分离 |
| 异步通讯 | 削峰、解耦 | 本地队列 / WorkManager |
| 幂等 | 一次与多次副作用相同 | 防重复点击 + 请求去重 |
| 服务状态 | 无状态优先；有状态外置 | 别在内存里存「全局用户会话」当唯一真相 |
| 补偿事务 | 正向失败则反向撤销 | 支付失败回滚库存 |
| 重试 | 有限次、退避、可取消 | 指数退避重试 |
| 熔断 | 失败率过高先切断 | 下游连续失败就停调，避免雪崩 |
| 限流 | 保护自己与下游 | 客户端防抖 / 服务端令牌桶 |
| 降级 | 牺牲非核心保核心 | 关闭推荐，只保留基础列表 |

**组合口诀**：

- 错误：`重试 + 熔断 + 幂等`  
- 一致性：强一致少用；最终一致配补偿  
- 流控：`限流 + 降级`  
- 运维：监控 + 流量调度 + 健康检查  

### 7.4 管理设计（工程基础设施）

- 分布式锁：慎用；优先用 DB 唯一约束 / 队列串行  
- 配置中心：动态开关、降级开关  
- 边车 / Service Mesh / 网关：流量治理外置  
- 部署升级：滚动、蓝绿、金丝雀（客户端灰度同理）  

### 7.5 性能设计

- 缓存（064）  
- 异步处理（065）：能异步的别同步堵请求路径  
- 数据库扩展（066）：读写分离、分库分表的前置条件与代价  
- 秒杀（067）：库存预减、排队、静态化——极端场景练手  
- 边缘计算（068）：把能靠近用户的计算/缓存前移（与 CDN、端侧缓存呼应）  

### 实践项目 E：简易秒杀 / 抢购

1. 预热库存到 Redis  
2. 接口限流 + 用户维度防刷  
3. MQ 异步下单；消费者幂等（订单号唯一）  
4. 下游支付超时：查询 + 补偿  
5. 熔断：支付服务失败率高时快速失败并降级提示  

**验收**：故障演练文档（kill 下游、打满 QPS、重复投递消息）+ 指标与改进。  
**出师口令**：部分失败与过载下，靠限流/熔断/幂等/补偿保住系统，并有演练证明。

### 专栏对照

- 020–027 分布式架构系列  
- 041–051 弹力设计全套  
- 058–063 管理设计  
- 064–068 性能设计  
- 081–085 练级攻略：分布式 / 微服务 / 容器运维

---

## 8. L5 — 架构进阶与「高手成长」

> 详解：[09-l5-architecture-and-growth.md](./09-l5-architecture-and-growth.md)（微服务决策、SLO/容量、领导力与沟通、项目 F）

从「能实现」到「能选型、能量化、能讲清演进」。高手路径是持续工作方式，不是结业证书。

### 8.1 微服务何时该上

专栏观点方向：**先单体模块化，再按边界拆**。客户端转后端常见踩坑：服务拆太碎，本地函数调用变成网络调用，复杂度爆炸。

判断清单：

- 是否有独立扩展 / 独立发布需求？  
- 团队边界是否清晰？  
- 是否具备观测、网关、配置、CI/CD？  
- 数据是否真的能拆（事务边界）？  

演进阶梯：模块化单体 → 异步边界 → 有证据再拆服务。

### 8.2 容量与 SLA

- 分清 SLI（度量）/ SLO（目标）/ SLA（对外承诺）  
- 做粗算：QPS × 单次成本 ≈ 资源；写明缓存命中率假设  
- 有预案：降级开关、限流阈值、扩容步骤；错误预算意识  

### 8.3 技术领导力与成长（专栏软技能）

- 005/006 技术领导力：用结果与影响力说话  
- 095–099 高效学习：源头、原理、知识地图、坚持实践  
- 100–105 高效沟通：Talk 与 Code 同等重要；契约变更书面化  
- 日常武器：ADR + 一页纸设计同步  

后端在组织里更常做「跨端契约 owner」——沟通与书面设计能力是硬技能。

### 实践项目 F：设计文档驱动的小型系统

选一个完整场景（例如「短链服务」或「内容审核流水线」），输出：

1. 需求与非功能需求（QPS、延迟、一致性）  
2. 架构图与数据模型  
3. 失败模式与弹力策略  
4. 观测与告警、容量粗算  
5. 演进路线（V1 单体 → V2 拆异步）+ ADR  

**出师口令**：能用一份设计文档说明目标、失败与演进，并推动相关方达成一致。

---

## 9. 推荐学习节奏（在职客户端）

| 阶段 | 周投入 | 重点 |
|------|--------|------|
| 第 1 月 | 8–10h | L0–L1：一个可上线的 CRUD + SQL |
| 第 2–3 月 | 8–10h | L2：缓存、索引、事务、Linux 排查 |
| 第 4–6 月 | 6–8h | L3–L4：MQ、弹力模式、一个完整压测项目 |
| 之后 | 持续 | L5：读论文/经典书、写设计文档、参与后端值班 |

### 每周固定动作

1. 读专栏 1–2 篇（先弹力/分布式，再练级攻略对应章）  
2. 写代码 ≥ 写笔记  
3. 用「客户端类比」复述一遍概念（教是最好的学）  
4. 周末做一次故障演练或压测小结

---

## 10. 书单与资料（精简版，对齐专栏取向）

### 必读/必翻

- 《程序员练级攻略》系列目录本身（按级别对照本文）  
- 《Designing Data-Intensive Applications》（数据密集型应用）——分布式与存储心智  
- 《UNIX 环境高级编程》/ Linux 基础——按需查  
- 数据库：MySQL / PostgreSQL 官方文档「事务与索引」章节  

### 论文与经典（L4+）

专栏 027/082 有分布式经典资料索引。优先顺序建议：

1. CAP / PACELC 相关综述（先建立权衡语言）  
2. Raft / Paxos 入门（理解共识为何难）  
3. 大厂工程博客：缓存、限流、秒杀实战  

### 正版专栏路径建议（后端向）

```
错误处理 011-012
→ 分布式 020-027
→ 弹力设计 041-051
→ 管理设计 058-063
→ 性能设计 064-068
→ 练级攻略 069-085（按需回填基础）
→ 面试与学习方法 091-099
```

---

## 11. 能力对照自检表

用勾选做季度回顾：

### 基础

- [ ] 能独立设计 REST/JSON API，并保证兼容演进  
- [ ] 能写正确事务与必要索引，会看执行计划  
- [ ] 能配置连接池、超时、重试，并说明为什么  

### 进阶

- [ ] 能设计缓存并处理穿透/击穿/雪崩  
- [ ] 能用 MQ 做异步与削峰，处理重复消费  
- [ ] 能解释并实现幂等、限流、熔断、降级之一整套  

### 架构

- [ ] 能画出系统依赖与故障传播路径  
- [ ] 能做简单容量估算与 SLO 定义  
- [ ] 能写清 ADR（架构决策记录）与回滚方案  

---

## 12. 结语

客户端转后端，最大的优势不是「会写界面」，而是你已经在极端约束下交付过产品。把同一套工程纪律，应用到**数据正确性、故障弹力与系统可观测**上，就是后端练级的主线。

《左耳听风》给的不是速成清单，而是：**高标准、重本质、靠实践**。按 L0→L5 走完一轮，你将不再只是「会调接口的客户端」，而是能与后端对等讨论设计、能独自扛一块服务的工程师。

下一篇：[02-interview-qa.md](./02-interview-qa.md) — 后端面试题与参考答案。  
完整可复制合集：[00-full-copy-pack.md](./00-full-copy-pack.md)。

---

<a id="part-11"></a>

# ========== PART 11：L0 HTTP 服务端视角 + Node 落地（11） ==========

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

---

<a id="part-12"></a>

# ========== PART 12：L0 Node.js 面试向详解（12） ==========

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

---

<a id="part-03"></a>

# ========== PART 03：L0 最小后端栈（03） ==========

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

---

<a id="part-04"></a>

# ========== PART 04：L1 入门与修养（04） ==========

# L1 — 后端入门与修养（4–8 周）详解

> 对应练级文档第 4 章。前置：已完成 [最小后端栈](./03-minimal-backend-stack.md)（能 Docker 起服务、有路由/配置/日志/healthz）。  
> L1 目标：**在共享数据与团队协作约束下，稳定交付带鉴权的业务接口**——从「能跑 Demo」到「能上预发」。

《左耳听风》把「程序员修养」放在专业基础之前：心态与规范决定你能走多远。客户端转后端，修养差距往往比语法差距更大。

---

## 0. L0 → L1：你在跨什么坎？

| | L0 最小栈 | L1 入门与修养 |
|--|-----------|----------------|
| 数据 | 可无 DB / 内存假数据 | 真关系数据库 + 迁移 |
| 协作 | 个人能跑 | PR、CI、可审查的代码 |
| 风险 | 挂了自己重启 | 坏数据、越权、不可逆迁移 |
| 交付物 | `/healthz` + ping | 多用户业务 API + 测试 |
| 心态 | 「我写通了」 | 「别人能接手、线上能回滚」 |

客户端类比：从「本地装包能用」到「提测、灰度、崩溃率、热修流程」——后端把同等纪律用在**数据和接口契约**上。

### 出师标准（8 周末自检）

- [ ] 能设计 3+ 张表，有主键/唯一约束/必要索引，会用 `EXPLAIN`  
- [ ] 写路径有事务；列表接口能扛一波简单压测不炸连接池  
- [ ] 有分层（handler / service / repo）与统一错误码  
- [ ] 鉴权后只能改自己的资源（垂直越权测过）  
- [ ] PR + CI（lint/test）跑通；DB 迁移可前进可回滚说明  
- [ ] 提问能带：复现步骤、request_id、相关日志、已排除项  

---

## 1. 程序员修养（转后端尤其重要）

专栏 072 强调：有修养的才可能成为工程师，否则容易停在「堆功能的码农」。下面按可操练项展开。

### 1.1 提问与排查

**坏提问：**「笔记接口 500 了，咋办？」

**好提问（模板）：**

```text
环境：staging / commit abc123 / 时间 UTC 12:05
现象：POST /v1/notes 返回 500，request_id=…
复现：用户 A，body={...}，curl 命令附上
已查：日志里 err=unique violation；本地同 body 可复现
已排除：Token 有效、DB 连接正常
问题：唯一约束是否应改成 upsert？还是前端防重？
```

习惯：

1. 先看日志 / 指标，再问人（对齐专栏：先自助）。  
2. 会写**最小复现**（最小 SQL / 最小 curl）。  
3. 区分：不会用工具 vs 设计决策需要讨论。

客户端映射：提 Bug 单要有机型、系统版、复现路径——后端换成 request_id 与环境。

### 1.2 安全意识（L1 必会的底线）

| 风险 | 做法 | 客户端类比 |
|------|------|------------|
| SQL 注入 | 参数化查询 / ORM，禁止字符串拼 SQL | 别拼 WebView URL |
| 越权 | 每个写操作校验 `resource.owner_id == current_user` | 别信客户端传来的 userId |
| 敏感日志 | Token、密码、手机号脱敏 | 崩溃日志勿带密钥 |
| 密钥管理 | 环境变量 / Secret，不进 git | keystore / 证书不进仓库 |
| 枚举攻击 | 登录统一文案、限流、ID 用不可猜 | 隐藏接口仍要鉴权 |

**越权自测（每次加资源接口都做）：**

1. 用户 A 创建资源，记下 id。  
2. 用户 B 的 Token 调 `PATCH/DELETE` 该 id → 必须 403/404。  
3. 列表接口不得泄露他人数据。

### 1.3 上线规范

后端「发版」不只是换包，常伴随**数据形态变化**。

最低规范：

1. **迁移可回滚或可前进修复**：写清 `up` / `down`（或 expand-contract 两步发布）。  
2. **兼容发布**：先加列可空 → 再双写/回填 → 再改代码依赖 → 最后收紧约束。  
3. **回滚预案**：镜像回滚命令、迁移是否需要反向、开关能否关掉新逻辑。  
4. **特性开关**：高风险逻辑可配置关闭（L1 用环境变量即可）。  

客户端映射：数据库迁移 ≈ 你做过的「本地 DB schema 升级」——只是现在失败会影响所有用户。

### 1.4 代码规范与分层

推荐三层（名字可变，边界要清）：

```
HTTP Handler / Controller   # 解析参数、鉴权上下文、写响应
    ↓
Service                      # 业务规则、事务边界、权限判断
    ↓
Repository / DAO             # SQL / ORM，不掺业务 if-else
```

统一错误码示例：

| code | HTTP | 含义 |
|------|------|------|
| `OK` | 200 | 成功 |
| `INVALID_ARGUMENT` | 400 | 参数错误 |
| `UNAUTHENTICATED` | 401 | 未登录 |
| `PERMISSION_DENIED` | 403 | 无权限 |
| `NOT_FOUND` | 404 | 不存在 |
| `ALREADY_EXISTS` | 409 | 冲突（唯一约束） |
| `INTERNAL` | 500 | 未知错误（日志详查） |

禁止：业务里 `catch` 后吞掉；Handler 里写 SQL；Service 里拼 HTML。

### 1.5 英文与文档

- 会查：Postgres/MySQL 文档、框架官方指南、少量 RFC（如 HTTP 语义）。  
- **OpenAPI / 接口文档当作产品交付**：字段、错误码、鉴权方式写清。  
- 变量/表名用一致英文；注释写「为什么」，少写「做什么」。

### 1.6 修养周练（穿插在 4–8 周里）

每周固定一次：

1. 给自己上周的一个 PR 写「若我是审查者会怎么骂」。  
2. 故意做一次越权测试并记结果。  
3. 读一篇官方文档（索引或事务），用自己的话写 10 行笔记。

---

## 2. 工程工具链

### 2.1 Git 工作流（专栏 031）

推荐起步：**主干 + Feature Branch + PR**。

```text
main ──────────────────────────●────
         \                    /
          feature/notes-crud ●
```

习惯：

- 小步提交，信息说清「为什么」  
- PR 描述：动机、怎么测、迁移/风险  
- Review 看：正确性、越权、索引、错误处理——不只看格式  
- 禁止直接 push main（可用 branch protection）

客户端映射：和你们 App 的 release 分支策略同构，只是后端更强调**迁移与回滚写在 PR 里**。

### 2.2 测试：单测 + 集成测

| 类型 | 测什么 | L1 最低覆盖 |
|------|--------|-------------|
| 单测 | Service 纯规则（权限、校验） | 创建/更新校验、越权拒绝 |
| 集成测 | HTTP + 真 DB（Testcontainers / Compose） | 登录 → CRUD → 越权 |

原则：

- 鉴权与核心**写路径**必须有自动化测试。  
- 测行为，不测实现细节（少断言「调用了某 mock 三次」）。  
- CI 红了不合并。

### 2.3 CI 流水线（最小）

每次 PR：

1. `lint` / format check  
2. `unit test`  
3. `integration test`（可 nightly 若太慢，但 L1 尽量 PR 内跑）  
4. `docker build`（确保镜像能编过）  

不必一上来就 CD 到生产；先做到「红灯不能合」。

### 2.4 本地依赖：Compose 起 MySQL/Postgres + Redis

在 L0 的 `docker-compose` 上扩展：

```yaml
services:
  api:
    build: .
    environment:
      DATABASE_URL: postgres://app:app@db:5432/notes?sslmode=disable
      REDIS_ADDR: redis:6379   # L1 可先不用，占位即可
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: notes
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d notes"]
      interval: 3s
      retries: 10
  # redis:
  #   image: redis:7-alpine
  #   ports: ["6379:6379"]
```

目标：`docker compose up` 后 API + DB 齐活，新人零手工装数据库。

### 2.5 连接池（L1 就要有意识）

```text
实例数 × 每实例最大连接  ≤  DB max_connections × 安全系数（如 0.7）
```

本地开发 2～10 个连接即可；压测时盯：等待连接超时、DB `too many connections`。  
客户端类比：OkHttp 连接池有上限——服务端对 DB 同理。

---

## 3. 关系数据库入门

> 四块详解（主键/唯一/外键/软删、索引与 EXPLAIN、事务、迁移工具）见  
> [05-relational-db-basics.md](./05-relational-db-basics.md)。

对齐专栏 080：先把**单库用正确**，再谈分库。

### 3.1 表设计基础

实践项目 B 的示意模型：

```text
users
  id PK
  email UNIQUE NOT NULL
  password_hash NOT NULL
  created_at

notebooks
  id PK
  user_id NOT NULL  → users.id
  title NOT NULL
  created_at
  UNIQUE(user_id, title)   -- 同一用户下笔记本名不重复

notes
  id PK
  notebook_id NOT NULL → notebooks.id
  user_id NOT NULL     -- 冗余 owner，方便鉴权与索引（可辩论）
  title NOT NULL
  body TEXT
  created_at
  updated_at
  UNIQUE(notebook_id, title)  -- 防重复标题
  INDEX(user_id, updated_at DESC)  -- 列表「我的笔记」
```

要点：

1. **主键**：自增或 UUID/ULID；团队统一即可。  
2. **唯一约束**：能靠 DB 防的并发问题，不要只靠 if。  
3. **外键**：学习阶段建议开着（教完整性）；高并发阶段再评估去掉与应用层保证。  
4. **软删除**：`deleted_at` 可空；唯一约束要注意「未删除行」——可用部分唯一索引。  
5. **时间**：存 UTC；展示层再转时区。

### 3.2 索引：直觉与验收

- B+Tree：适合等值、范围、排序；理解「有序树 + 回表」。  
- **最左前缀**：`(user_id, updated_at)` 可支持 `WHERE user_id=? ORDER BY updated_at`；单用 `updated_at` 往往用不上该联合索引。  
- **覆盖索引**：查询列都在索引里可少回表（进阶）。  
- **不是越多越好**：拖慢写入、占空间。

**每次列表接口上线前：**

```sql
EXPLAIN ANALYZE
SELECT id, title, updated_at
FROM notes
WHERE user_id = $1 AND deleted_at IS NULL
ORDER BY updated_at DESC
LIMIT 20;
```

验收：走 index scan / index only scan，而不是大表 seq scan（数据量起来后）。

### 3.3 事务：ACID 与隔离级别（够用版）

| 概念 | 一句话 |
|------|--------|
| Atomicity | 一组写要么全成要么全不成 |
| Consistency | 不破坏约束与业务不变量 |
| Isolation | 并发事务如何互不干扰 |
| Durability | 提交后掉电不丢 |

常见隔离级别：读未提交 → 读已提交 → 可重复读 → 串行化。  
MySQL InnoDB 默认可重复读；Postgres 默认读已提交。L1 先做到：

- **写路径显式事务**：创建笔记 + 更新笔记本计数（若有）同事务。  
- 捕获唯一约束冲突 → 返回 `ALREADY_EXISTS`，不要变 500。  
- 长事务禁止：事务里调外部 HTTP。

### 3.4 迁移工具

| 语言栈 | 常用工具 |
|--------|----------|
| Go | golang-migrate / atlas |
| Java/Kotlin | Flyway / Liquibase |
| Node | Prisma migrate / node-pg-migrate / Knex |

规范：

1. 迁移文件版本化进 git。  
2. 应用启动或 CI 执行 migrate（团队选一种，别混用手工改表）。  
3. PR 里贴 `up` SQL，并说明回滚策略。  
4. 禁止在生产手工点改表却不落迁移文件。

### 3.5 密码与鉴权（项目 B 需要）

- 密码：**argon2id / bcrypt** 哈希，禁止明文、禁止可逆加密当「哈希」。  
- Session 或 JWT 二选一（见面试题 A3）；L1 用 JWT 也可，但要短过期。  
- 所有业务 API：`Authorization: Bearer ...`；`/healthz` 除外。

---

## 4. 实践项目 B：多用户笔记服务

在 L0 最小栈上演进，不要重开毫无规范的玩具项目。

### 4.1 功能范围

1. 注册 / 登录，返回 Token。  
2. Notebook CRUD（仅本人）。  
3. Note CRUD（仅本人；隶属自己的 notebook）。  
4. 笔记列表：分页（`limit` + `cursor` 或 `offset`）、按 `updated_at` 排序。  
5. OpenAPI 文档或至少 README 接口表。

### 4.2 接口草图

```text
POST   /v1/auth/register
POST   /v1/auth/login
GET    /v1/notebooks
POST   /v1/notebooks
GET    /v1/notebooks/{id}/notes
POST   /v1/notebooks/{id}/notes
PATCH  /v1/notes/{id}
DELETE /v1/notes/{id}
GET    /v1/notes?limit=20&cursor=...
```

### 4.3 必做验收（勾完才算 L1 出师）

**正确性**

- [ ] 唯一约束冲突返回 409，不 500  
- [ ] B 用户无法改 A 的笔记  
- [ ] 删除 notebook 的策略明确（级联 / 禁止非空删除）并有测试  

**数据与性能**

- [ ] 列表 SQL 有 `EXPLAIN` 截图或文字结论进 PR  
- [ ] 用 k6/wrk 打 `GET /v1/notes`：观察 P95、错误率、DB 连接数  
- [ ] 连接池参数写进配置，不是硬编码魔法数  

**工程**

- [ ] Compose 一键：api + db  
- [ ] CI 绿；核心写路径有集成测  
- [ ] 至少 1 个迁移文件 + 回滚说明  
- [ ] 结构化日志含 `request_id`、`user_id`  

### 4.4 简易压测示例（k6）

```javascript
import http from "k6/http";
import { check } from "k6";

export const options = { vus: 20, duration: "30s" };

export default function () {
  const res = http.get("http://localhost:8080/v1/notes?limit=20", {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(res, { "status 200": (r) => r.status === 200 });
}
```

看：是否出现 5xx、延迟尖刺、Postgres 日志慢查询。先学「观察」，不追求刷高 QPS。

---

## 5. 建议排期（4–8 周，在职每周 8–10h）

### 紧凑版（4 周）

| 周 | 主题 | 产出 |
|----|------|------|
| 1 | 修养 + Git/CI + Compose 加 DB | 空库可连；CI 跑通；错误码表 |
| 2 | 表设计 + 迁移 + 注册登录 | users 表；JWT/Session；集成测登录 |
| 3 | Notebook/Note CRUD + 越权 | 分层清晰；越权测试全绿 |
| 4 | 索引 + EXPLAIN + 压测 + 文档 | 项目 B 验收清单打勾；补 OpenAPI |

### 充裕版（6–8 周）

在紧凑版上增加：

- 软删除与部分唯一索引  
- 游标分页（代替粗糙 offset）  
- 密码重置 / 刷新令牌其一  
- 读一篇事务隔离深度文章并做笔记  
- 给同学做一次 30 分钟项目演示（练沟通，对齐专栏）

### 每周固定节奏

1. **1 个小功能竖切**（接口通 + 测试 + 迁移）  
2. **1 次越权或安全自测**  
3. **0.5 次文档/RFC/专栏阅读**（072 / 080 / 031）  
4. 周末：压测或重构分层，不新开需求  

---

## 6. 客户端背景：L1 最易踩的坑

| 坑 | 为什么会踩 | 纠正 |
|----|------------|------|
| 信客户端传的 `userId` | App 里一直自己带 uid | 只信 Token 解析出的身份 |
| 先查再改无事务 | 本地 DB 单线程思维 | 用事务或 `UPDATE ... WHERE` 原子条件 |
| 无唯一约束靠 if | UI 已防重复点击 | 并发下 if 会穿；DB 约束兜底 |
| 列表无索引 | 数据少时感觉很快 | 万级数据后炸；用 EXPLAIN |
| 迁移手改库 | 像改本地 SQLite | 必走迁移文件 |
| 测试只点点点 | 客户端很多手工回归 | 核心写路径自动化 |

---

## 7. 专栏与延伸阅读

| 主题 | 《左耳听风》 | 补充 |
|------|--------------|------|
| 修养 | 072 | 团队规范、Code Review 文化 |
| Git | 031 | 选一种工作流吃透 |
| 数据库 | 080 | 官方文档：索引与事务章节 |
| 错误处理 | 011–012 | 与统一错误码一起做 |
| 面试态度 | 091–094 | 用项目 B 当故事储备 |

正版专栏：[左耳听风](https://time.geekbang.org/column/intro/100002201)

---

## 8. L1 结束后接到哪里？

L1 出师后进入 [01-leveling-guide.md](./01-leveling-guide.md) 的 **L2 专业基础**：

- 缓存（Redis）接在笔记列表上  
- Linux / 网络排查  
- 更系统的数据结构直觉  

面试题对照：[02-interview-qa.md](./02-interview-qa.md) 的 **B 组（数据库）**、**A 组（鉴权）**——用项目 B 的真实细节作答，勿背空定义。

---

## 9. 一页纸检查表（可打印）

```text
修养
  [ ] 提问带 request_id 与复现
  [ ] 越权测试通过
  [ ] 密钥未进仓库；日志已脱敏
  [ ] 迁移有回滚说明

工程
  [ ] Feature Branch + PR
  [ ] CI：lint + test + docker build
  [ ] Compose：api + db

数据
  [ ] 主键 / 唯一约束 / 必要索引
  [ ] 写路径有事务
  [ ] EXPLAIN 过列表查询
  [ ] 连接池有上限意识

项目 B
  [ ] 注册登录 + 三表 CRUD
  [ ] 只能改自己的资源
  [ ] 压测看过一轮
  [ ] 接口文档可读
```

---

<a id="part-05"></a>

# ========== PART 05：L1 关系数据库基础（05） ==========

# 关系数据库基础详解（L1）

> 展开练级文档 / L1 中的四块：**表设计、索引与 EXPLAIN、事务、迁移工具**。  
> 对齐《左耳听风》数据库取向：先把单库用正确，再谈分库分表。  
> 客户端类比贯穿全文：你熟悉的 SQLite / Realm schema，搬到「多连接并发写」的共享数据库。

---

## 0. 为什么 L1 要把数据库学「死」一点？

客户端本地库通常：

- 单用户、写冲突少  
- 坏了可以清数据重装  

后端库是：

- 多实例、多请求同时写同一行  
- 坏数据可能不可逆、影响全站  

所以 L1 不追求「会写 SQL」，而追求：**约束兜底并发、索引撑住列表、事务守住不变量、迁移可审计**。

---

## 1. 表设计：主键、唯一约束、外键、软删除

### 1.1 主键（Primary Key）

**作用：** 唯一标识一行；被其他表引用；InnoDB/成簇索引场景下还影响物理顺序。

| 方案 | 优点 | 缺点 | 何时用 |
|------|------|------|--------|
| 自增 BIGINT | 短、有序、索引友好 | 扩缩容/分库麻烦；易被枚举猜 ID | 单库学习与多数业务起步 |
| UUID | 全球唯一、可客户端生成 | 乱序写入、索引膨胀（可用 UUIDv7 缓解） | 合并数据、暴露给外部前再映射 |
| ULID / KSUID / UUIDv7 | 趋势有序 + 可分布生成 | 生态略新，团队要统一 | 想兼顾有序与分布式 |

**实践建议（L1）：**

- 团队选一种，全项目一致。  
- 对外暴露的 ID 可与内部 PK 分离（可选）：内部自增，对外发 ULID，防枚举。  
- 主键不要业务含义（别用手机号当 PK）。

客户端类比：Core Data / Room 的 `@PrimaryKey`——后端还要考虑「别人猜 ID 来越权」。

### 1.2 唯一约束（UNIQUE）

**作用：** 在数据库层保证「不能重复」，比应用里 `if (exists) return` 可靠。

```sql
-- 全局唯一
UNIQUE (email)

-- 作用域唯一：同一用户下笔记本名不重复
UNIQUE (user_id, title)
```

并发下两个请求同时「先查再插」都会以为不存在 → 都插入成功；**只有唯一约束能把其中一个打成冲突**。  
应用层应把冲突翻译成 `409 ALREADY_EXISTS`，而不是 500。

**和幂等的关系：** 订单号 / Idempotency-Key 建唯一索引，是最便宜的幂等实现之一。

### 1.3 外键策略（Foreign Key）

外键声明：「子表这列的值必须在父表存在」。

```sql
notebook_id BIGINT NOT NULL
  REFERENCES notebooks(id)
  ON DELETE RESTRICT   -- 或 CASCADE / SET NULL
```

| 策略 | 含义 | 典型场景 |
|------|------|----------|
| `RESTRICT` / `NO ACTION` | 有子行则不让删父行 | 笔记本下还有笔记 |
| `CASCADE` | 删父则级联删子 | 强归属、可一起消失 |
| `SET NULL` | 父删后子外键置空 | 可选关联 |

**开不开外键？**

| 阶段 | 建议 |
|------|------|
| L1 学习 / 中小项目 | **建议开**：强制你想清楚归属与删除语义 |
| 超高写入、跨库、大量批量任务 | 常关掉外键，改应用层保证 + 异步校验 |

代价：写入多一次检查；跨库无法用；错误的 `CASCADE` 可能误删一大片。

客户端类比：SQLite 也可开 FK；移动端常懒得开，后端开着能少很多脏数据。

### 1.4 软删除（Soft Delete）

硬删除：`DELETE FROM notes WHERE id=?`  
软删除：`UPDATE notes SET deleted_at = now() WHERE id=?`，查询加 `WHERE deleted_at IS NULL`。

**优点：** 可恢复、审计友好、部分合规场景好做。  
**坑：**

1. **所有查询都要记得过滤** `deleted_at IS NULL`（易漏）。  
2. **唯一约束冲突**：删了标题叫「日记」的笔记后，再建同名会撞唯一索引。  

Postgres 解法（部分唯一索引）：

```sql
CREATE UNIQUE INDEX notes_notebook_title_live_uidx
  ON notes (notebook_id, title)
  WHERE deleted_at IS NULL;
```

MySQL 8+ 可用函数/生成列等技巧，或「删除时改 title 加后缀」的土办法——团队要显式约定。

3. 列表索引常要带上 `deleted_at` 条件，设计索引时一起考虑。

### 1.5 表设计检查清单

- [ ] 每表有明确 PK  
- [ ] 业务不变量能写成 UNIQUE / CHECK 的，就写成约束  
- [ ] 删除语义：硬删 / 软删 / 级联，写进设计说明  
- [ ] 时间戳 UTC；`created_at` / `updated_at`  
- [ ] 不把「当前用户」只靠客户端字段，owner 在服务端写入  

---

## 2. 索引：B+Tree、最左前缀、覆盖索引、EXPLAIN

### 2.1 B+Tree 直觉（够用版）

把索引想成**排好序的目录**：

- 等值查找：`WHERE user_id = 42`  
- 范围查找：`WHERE updated_at > ...`  
- 排序：`ORDER BY updated_at DESC`（若顺序与索引一致可免额外 sort）  

叶子节点存「键 → 行指针（或主键）」；拿到指针再回表取完整行。

**不适合：** 很低区分度的列单独建索引（如只有 0/1 的 flag），优化器可能直接全表扫更划算。

客户端类比：通讯录按姓排序后二分查找；没有目录就从头翻。

### 2.2 最左前缀

联合索引 `(user_id, updated_at)` 可以很好支持：

```sql
WHERE user_id = ?
ORDER BY updated_at DESC

WHERE user_id = ? AND updated_at > ?
```

通常**不能很好**支持：

```sql
WHERE updated_at > ?          -- 没用到最左列 user_id
ORDER BY updated_at           -- 同上
```

口诀：**联合索引从左到右用；中间不能跳着指望（优化器有时仍能用，但别赌）。**

建索引前先写下真实 `WHERE` + `ORDER BY`，再决定列顺序：等值列在前，范围/排序列在后。

### 2.3 覆盖索引（Covering Index）

若 `SELECT` 的列**全部**落在索引里，引擎可能 **Index Only Scan**，少回表。

```sql
-- 索引 (user_id, updated_at) 若还 INCLUDE/额外包含 title
-- 或建 (user_id, updated_at, title) —— 权衡写入成本
SELECT id, title, updated_at FROM notes WHERE user_id = ? ORDER BY updated_at DESC;
```

注意：主键在 InnoDB 二级索引叶子常已带上，细节因引擎而异；用 `EXPLAIN` 看是否 `Using index` / `Index Only Scan`。

L1 不要求凡事覆盖索引，但要知道：**列表若只取少数字段，有机会靠索引加速。**

### 2.4 列表接口上线前跑 EXPLAIN

把下面当成发布门禁：

```sql
EXPLAIN ANALYZE
SELECT id, title, updated_at
FROM notes
WHERE user_id = 123 AND deleted_at IS NULL
ORDER BY updated_at DESC
LIMIT 20;
```

| 想看到 | 警惕 |
|--------|------|
| Index Scan / Index Only Scan | Seq Scan 且 rows 巨大 |
| rows 估算与 LIMIT 同量级 | 扫百万行再 limit 20 |
| 执行时间稳定 | 临时文件 sort、巨大 cost |

流程建议：

1. 本地灌 1万～10万行假数据再 EXPLAIN（空表什么都「很快」）。  
2. 截图或文本贴进 PR。  
3. 缺索引就补迁移，再测一遍。

### 2.5 索引成本

每加一个索引：

- 写入变慢（要维护树）  
- 占磁盘  
- 优化器选择变复杂  

原则：**没有慢查询证据，不提前堆索引**；有 `WHERE` 形态再针对性加。

---

## 3. 事务：ACID、隔离级别、写路径纪律

### 3.1 ACID（面试能说人话）

| 字母 | 含义 | 工程体感 |
|------|------|----------|
| **A** Atomicity | 原子 | 转账扣款+入账，不能只成一半 |
| **C** Consistency | 一致 | 不破坏约束与业务不变量（约束是 C 的硬保证） |
| **I** Isolation | 隔离 | 并发事务互不「脏」干扰的程度可调 |
| **D** Durability | 持久 | 提交成功后宕机也不该丢（刷盘/复制另说） |

### 3.2 隔离级别与读异常

| 级别 | 脏读 | 不可重复读 | 幻读 | 常见默认 |
|------|------|------------|------|----------|
| 读未提交 | 可能 | 可能 | 可能 | 极少用 |
| 读已提交 | 否 | 可能 | 可能 | PostgreSQL |
| 可重复读 | 否 | 否 | 视实现 | MySQL InnoDB |
| 串行化 | 否 | 否 | 否 | 最严、最慢 |

不必死背表格，要能举例：

- **脏读：** 读到别人未提交的修改，对方一回滚你就读了鬼数据。  
- **不可重复读：** 同一事务两次读同一行，中间被别人改了。  
- **幻读：** 两次范围查询，中间多出/少了行。

L1 务实策略：用数据库默认隔离级别；对「库存扣减」等用**原子 SQL** 或乐观锁，而不是一上来全局串行化。

### 3.3 写路径显式事务

把「必须一起成功」的步骤包进同一事务：

```text
BEGIN
  插入 note
  更新 notebook.note_count += 1
  （校验 notebook 属于当前用户）
COMMIT
```

伪代码（示意）：

```go
tx, err := db.BeginTx(ctx, nil)
// ... create note
// ... bump notebook count
// 任一步失败 → Rollback；全成功 → Commit
```

**显式**的意思：不要靠「每个语句自动提交」拼出半成品状态。

### 3.4 事务内禁止调外部 HTTP（铁律）

反例：

```text
BEGIN
  扣库存
  调用支付中心 HTTP   ← 可能 2s～30s，连接被占住
  写订单号
COMMIT
```

危害：

1. **长事务**锁行/占连接，池被掏空（回到「连接池总账」）。  
2. HTTP 超时后 DB 事务状态难料理（到底扣没扣？）。  
3. 外部系统成功、本地回滚 → 两边不一致。  

正例：

```text
本地事务只做本地：写「待支付」订单 + 占库存（或预留）
COMMIT
再调支付
根据回调/查询结果，再开短事务更新状态（或走消息+补偿）
```

客户端类比：不要在持有 DB 锁时做网络；就像不要在主线程同步打大网——后端是「别占着连接池做 IO」。

### 3.5 事务小技巧

- 事务尽量短：先算好数据，进事务只写。  
- 捕获唯一冲突 → 业务错误，不 500。  
- `SELECT FOR UPDATE` 能用但吞吐差，优先 `UPDATE ... WHERE stock >= 1`。  
- 上下文带 timeout：`ctx` 超时要能取消，避免事务悬挂（驱动与 DB 需支持）。

---

## 4. 迁移工具：Flyway / Liquibase / golang-migrate

### 4.1 为什么要迁移工具？

禁止：「我在生产 Navicat 点了个新列」。

迁移工具让 schema 变更变成：

1. **版本化文件**进 git  
2. **可重复执行**于 local / CI / staging / prod  
3. **有执行历史表**（如 `flyway_schema_history` / `schema_migrations`）  
4. 与发版流程绑定，可审查  

客户端类比：Room / Core Data / Realm 的 Migration——后端是团队共享的同一份演进史。

### 4.2 三选一对照

| 工具 | 生态 | 迁移形态 | 特点 |
|------|------|----------|------|
| **Flyway** | Java/Kotlin/Spring 主流 | 默认 SQL（也有 Java） | 简单直给；版本号 `V1__create_notes.sql` |
| **Liquibase** | Java 企业常见 | XML/YAML/JSON/SQL | 变更集丰富；回滚与条件迁移表达力强 |
| **golang-migrate** | Go 主流 | 成对 `up.sql` / `down.sql` | 轻；CLI + 库都能跑 |

Node 常见：Prisma Migrate、Knex、node-pg-migrate——思想相同。

**怎么选：** 跟主语言走即可；原则比工具名重要。

### 4.3 文件与命名示例

**Flyway：**

```text
db/migration/
  V1__create_users.sql
  V2__create_notebooks_and_notes.sql
  V3__add_notes_user_updated_index.sql
```

**golang-migrate：**

```text
db/migrations/
  000001_create_users.up.sql
  000001_create_users.down.sql
  000002_create_notes.up.sql
  000002_create_notes.down.sql
```

`up`：前进；`down`：回滚（能写则写；不能安全 down 的要在 PR 说明「只前进修复」）。

### 4.4 安全演进：Expand / Contract

大变更不要「一步改死」：

1. **Expand：** 加新列（可空）/ 新表，旧代码仍跑  
2. **Migrate data：** 回填数据，双写过渡  
3. **Switch：** 发布读新列的代码  
4. **Contract：** 删旧列、加 NOT NULL（确认无旧代码依赖后）  

这样回滚应用版本时，schema 仍然兼容。

### 4.5 团队规范（抄进 README）

1. 任何 schema 变更必须提迁移文件，禁止手工改生产表。  
2. PR 必含：`up` SQL、风险、是否锁表、回滚策略。  
3. CI 对空库跑一遍 migrate + 测试。  
4. 生产迁移：窗口、备份、锁表预估（加索引在大表上可能久，考虑 `CONCURRENTLY` 等，按引擎文档）。  
5. 一人申请执行、一人复核（简单的 four-eyes）。

### 4.6 和连接池 / 发布的关系

- 迁移任务单独跑（Job / init container），**不要**每个 API 副本同时抢着 migrate（要有锁；Flyway/migrate 通常有，仍建议单职务执行）。  
- 迁移占用连接，算进总账与发布窗口。  
- 失败要停发布，不要假装「应用先上、表以后再说」。

---

## 5. 串起来：从设计到上线的一条龙

```text
1. 写出表与约束（PK / UNIQUE / FK / 软删策略）
2. 写出列表 SQL → 设计索引列顺序
3. 灌数据 EXPLAIN → 调索引
4. 写路径用短事务；禁止事务内 HTTP
5. 变更写成 Flyway / migrate 文件
6. CI 迁移 + 测试 → 预发验证 → 生产迁移 → 放开流量
```

---

## 6. 动手练习（半天～一天）

1. 为笔记列表建错一个「只有 `updated_at`」的索引，EXPLAIN 看是否仍 Seq Scan。  
2. 改成 `(user_id, updated_at)`，再 EXPLAIN 对比。  
3. 两个并发请求插相同 `(notebook_id, title)`，确认一个成功一个唯一冲突。  
4. 写一个 `up`：加 `deleted_at` + 部分唯一索引；写 `down` 或说明不可 down 的原因。  
5. 故意在事务里 `sleep` 5 秒（模拟 HTTP），看连接池是否被占满。

---

## 7. 面试时可怎么答（浓缩）

- **唯一约束：** 并发下应用 if 不可靠，用不变量落在 DB。  
- **最左前缀：** 联合索引列顺序对齐 WHERE/ORDER BY。  
- **EXPLAIN：** 列表上线门禁，避免线上才发现全表扫。  
- **事务内 HTTP：** 长事务占连接、难回滚，拆成本地事务 + 异步/回调。  
- **迁移工具：** schema 当代码版本管理，可重复、可审查。

更完整题库见 [02-interview-qa.md](./02-interview-qa.md) B 组。

---

## 8. 相关文档

- [04-l1-entry-and-craft.md](./04-l1-entry-and-craft.md) — L1 总览与项目 B  
- [01-leveling-guide.md](./01-leveling-guide.md) — 练级主文档  
- 专栏：080 数据库；011/012 错误处理（约束冲突如何返回）

---

<a id="part-06"></a>

# ========== PART 06：L2 专业基础（06） ==========

# L2 — 专业基础（2–3 个月）详解

> 对应练级文档第 5 章。前置：L1 出师（真库、鉴权、迁移、越权测试过关）。  
> L2 目标：具备**系统视角 + 复杂度直觉 + 缓存工程能力**，能解释服务慢在哪、缓存怎么加才不埋雷。  
> 对齐《左耳听风》专业基础篇与性能设计「缓存」；客户端转后端按本文优先级学，不必按学院课表重读四年。

---

## 0. L1 → L2：跨的是什么坎？

| | L1 | L2 |
|--|----|----|
| 重心 | 把接口做对、数据做对 | 把接口做快、故障可解释 |
| 数据 | 单库 + 索引 | DB + **Redis 缓存** |
| 排障 | 看应用日志 | 还看 TCP/连接/FD/线程池 |
| 思维 | CRUD 正确性 | **复杂度、容量、命中率** |
| 产出 | 笔记服务 | 热点列表带缓存 + 一次压测对比报告 |

### 出师标准（2～3 月末自检）

- [ ] 能画清：客户端 → LB → 服务 → DB/Redis 的超时与连接关系  
- [ ] 能用 `ss`/`lsof`/`top`/`EXPLAIN` 之一解释「慢」或「打满」  
- [ ] 说清哈希/跳表/堆/树各自「快在哪」，并对应到 Redis 或索引  
- [ ] 实现 Cache-Aside；能防穿透/击穿/雪崩的至少各一种手段  
- [ ] 写后删缓存；能量化：加缓存前后 P95 / DB QPS 变化  

---

## 1. 系统与网络（必补）

客户端已会调 HTTP；L2 要懂**连接与资源**在服务端如何被消耗。

### 1.1 TCP：后端天天踩的几件事

| 概念 | 你要留下的印象 | 工程关联 |
|------|----------------|----------|
| 三次握手 / 四次挥手 | 新建连接贵 | 连接池、Keep-Alive 有意义 |
| 滑动窗口 / 拥塞控制 | 吞吐受限于双方与网络 | 「带宽够却慢」可能是窗口/丢包 |
| `TIME_WAIT` | 主动关闭方会短暂占端口 | 短连接风暴、端口耗尽 |
| 连接复用 | 握手摊销到多次请求 | HTTP Keep-Alive、连接池 |

**超时预算（务必建立）：**

```text
客户端 10s
  └─ 网关 3s
       └─ 服务 2s
            ├─ Redis 100ms
            └─ DB 500ms
```

层层更短，避免客户端已放弃、服务端还在占着连接算（呼应 L1「事务内禁止 HTTP」与连接池总账）。

### 1.2 HTTP/1.1、HTTP/2、反向代理

- **HTTP/1.1 Keep-Alive**：同一 TCP 上串行多个请求；队头阻塞仍在。  
- **HTTP/2**：多路复用，同连接并行流；后端与网关常已终结 TLS/H2。  
- **反向代理（Nginx / Envoy / 云 LB）**：TLS 终结、负载均衡、限流、缓冲；服务多半在内网说 HTTP/1.1。

客户端映射：OkHttp 连接池 ≈ 服务访问 Upstream 的 Keep-Alive 池；你调 API 的「慢」，可能在代理缓冲或上游连接耗尽。

### 1.3 进程 / 线程 / 协程与 IO 模型

| 模型 | 特点 | 后端体感 |
|------|------|----------|
| 多进程 | 隔离好，切换重 | Nginx worker、部分 PHP |
| 多线程 | 共享内存，要注意锁 | Java 经典线程池 |
| 协程（goroutine 等） | 轻量并发，适合高并发 IO | Go 默认心智 |
| 阻塞 IO | 一连接一线程易打满 | C10K 问题来源 |
| 非阻塞 / 多路复用 | 少数线程盯很多 FD | epoll、Netty、Node、Go runtime |

专栏 077/078 的取向：理解 **C10K**——瓶颈常在「线程/连接模型」，不在业务 if-else。

**客户端映射：** ANR ≈ 服务端工作线程池被慢 SQL/下游拖满，新请求排队到超时。

### 1.4 Linux 排查最小工具箱

不必成为运维，但要会这几下：

| 命令 | 看什么 |
|------|--------|
| `ps` / `top` / `htop` | CPU、内存、谁在忙 |
| `ss -lntp` / `ss -s` | 监听端口、连接状态统计 |
| `lsof -p <pid>` | 打开了哪些 FD（文件/套接字） |
| `ulimit -n` | 进程可打开 FD 上限 |
| `df` / `free` | 磁盘与内存是否见底 |
| 应用指标 | 线程池队列、DB/Redis 池等待 |

练习：把服务压到变慢，用上面工具说清是 **CPU 满、连接满、还是池等待**。

### 1.5 本节一周练法

1. 画一张自己项目的超时与连接图。  
2. 用 `wrk`/`k6` 打接口，同时 `ss -s` 看连接变化。  
3. 故意把 DB 池调极小，观察错误形态（超时 vs too many connections）。

---

## 2. 数据结构与算法（够用即可，定向加深）

目标不是刷题运动员，而是：**看到 Redis / 索引 / 限流时，知道底下为什么是 O(1)/O(log n)**，并避免在请求路径写出事故级复杂度。

### 2.1 四件套：哈希、跳表、堆、树

#### 哈希（Hash）

- **操作：** 平均 `O(1)` 放入 / 查找（冲突多时退化）。  
- **用在哪：** Redis `GET/SET`（SDS + dict）、HashMap、布隆过滤的位图哈希。  
- **体感：** 按 key 取缓存快，是因为哈希定位桶，不是扫全表。  
- **注意：** 最坏冲突、扩容、热 key 打在同一分片。

#### 跳表（Skip List）

- **操作：** 期望 `O(log n)` 查找 / 范围。  
- **用在哪：** Redis `ZSET`（有序集合：排行榜、延迟队列近似）。  
- **体感：** 既要按 member 找，又要按 score 排序范围取，跳表层叠索引实现。  
- **对比：** 也可以用平衡树；Redis 选跳表实现简单、范围友好。

#### 堆（Heap / 优先队列）

- **操作：** 取最大/最小 `O(1)`，插入/删除 `O(log n)`。  
- **用在哪：** 定时任务、TopK、合并有序流、部分调度器。  
- **体感：** 「总要下一个最早到期的任务」用堆，别每次排序全表。

#### 树（尤其 B+Tree）

- **操作：** `O(log n)` 查找；范围/排序友好。  
- **用在哪：** MySQL/PG **索引**（你已在 L1 学过叶子与回表）。  
- **体感：** 索引快是因为树高矮、有序，不是魔法。

| 结构 | 复杂度直觉 | 后端落点 |
|------|------------|----------|
| 哈希 | 平均 O(1) | Redis String/Hash、本地缓存 |
| 跳表 | O(log n) + 范围 | Redis ZSET |
| 堆 | Top/极值快 | 优先队列、调度 |
| B+Tree | O(log n) + 范围 | DB 索引 |

### 2.2 复杂度直觉：请求路径上的红线

一次 API 请求的预算往往是 **几十～几百毫秒**。在路径上：

| 写法 | 复杂度 | 1万元素时 | 态度 |
|------|--------|-----------|------|
| 按主键 / 索引查一行 | ~O(log n) 或 O(1) | 可接受 | 正常 |
| 全表扫再过滤 | O(n) | 危险 | 要索引或限制 |
| 双重循环对每条再查库 | O(n²) 或 n 次 RTT | 事故 | **禁止上线** |
| 在内存把结果排一次 | O(n log n) | 看 n | n 大交给 DB/缓存 |

**典型事故（N+1）：**

```text
list = SELECT * FROM notebooks WHERE user_id=?   -- 返回 50 本
for each notebook:
    SELECT * FROM notes WHERE notebook_id=?     -- 又 50 次查询
```

改为 Join、一次 `IN (...)`、或缓存聚合结果。

**客户端映射：** 主线程里双重循环绑 View 会卡顿；服务端同等写法会拖垮 DB 与线程池。

### 2.3 刷题怎么服务后端（而不是反过来）

- 面试：LeetCode **Medium 为主**，数组/哈希/双指针/树/堆/二分足够多数岗位。  
- 工作：把题感迁移到：  
  - **限流：** 滑动窗口计数、令牌桶  
  - **缓存淘汰：** LRU（哈希 + 双向链表）  
  - **调度：** 堆  
  - **排行榜：** ZSET / 跳表思维  

建议节奏：每周 3～5 题 + 每两周把一题对应到「我服务里的一个点」。

### 2.4 本节自测题（口述）

1. 为什么 Redis GET 快？最坏会怎样？  
2. 排行榜为什么常用 ZSET 而不是每次 SQL `ORDER BY`？  
3. 什么是 N+1？怎么改？  
4. 二级索引叶子有什么？和哈希索引适用场景差在哪？

---

## 3. 缓存（L2 分水岭）

对齐专栏 064：缓存是性能设计第一课，也是最容易「加了更抽」的地方。

### 3.1 本地缓存 vs 分布式缓存

| | 本地（进程内） | 分布式（Redis 等） |
|--|----------------|-------------------|
| 速度 | 最快（无网络） | 快（一次 RTT） |
| 一致性 | 多实例各有一份，易不一致 | 共享一份，相对好管 |
| 容量 | 受单机内存限 | 可扩展 |
| 适用 | 极热、允许短暂不一致的配置/字典 | 会话、热点列表、分布式锁备选 |

常见：**本地 + Redis** 二级；L2 先把 **Redis Cache-Aside** 做稳。

客户端映射：Memory + Disk 图片缓存；淘汰策略 ≈ LRU/LFU。

### 3.2 Cache-Aside（旁路缓存）——最常用

**读：**

```text
1. 读 Redis
2. 命中 → 返回
3. 未命中 → 读 DB → 写入 Redis（带 TTL）→ 返回
```

**写：**

```text
1. 写 DB 成功
2. 删除 Redis 对应 key（推荐删，而不是更新）
3. 下次读自动回源重建
```

为什么写后常**删**不**改**缓存：并发下「更新缓存」容易留下脏值；删 key 更简单，代价是下次多一次回源。

### 3.3 穿透、击穿、雪崩

| 问题 | 现象 | L2 至少会一种防法 |
|------|------|-------------------|
| **穿透** | 查不存在的数据，每次打到 DB | 缓存空值短 TTL；参数校验；布隆过滤（进阶） |
| **击穿** | 某个热点 key 过期瞬间大量回源 | 互斥重建（SET NX）；逻辑过期；永不过期 + 异步刷新 |
| **雪崩** | 大量 key 同时过期或 Redis 挂 | TTL 加随机抖动；多级缓存；熔断降级（回源限流） |

### 3.4 TTL 与一致性窗口

- TTL：太长 → 脏得久；太短 → 命中率低。列表类可 30s～5min + **随机 0～30s** 防同时过期。  
- 写后删缓存仍有极短不一致窗口（并发读写）；多数业务可接受；强一致场景要换方案（延迟双删、订阅 binlog 等，L4 再深挖）。  
- **用日志验证：** 打 `cache_hit` / `cache_miss` / `cache_delete` 计数，看窗口行为。

### 3.5 Key 设计规范

```text
app:notes:list:{user_id}     # 业务:实体:场景:主键
app:note:{note_id}
```

- 可预测、可批量删（注意 Redis 集群 slot；删模式慎用 `KEYS`）。  
- 值：JSON 或压缩；大 value 要警惕（热 key、网络带宽）。

### 3.6 缓存不是银弹

不要缓存：

- 极低频又极大的数据  
- 强实时强一致且写极多的数据（或只缓存明确可脏的部分）  
- 未想清楚淘汰与失效的「随手 set」  

先 EXPLAIN 优化 SQL，再决定是否缓存——**索引能解决的，别一上来甩 Redis**。

---

## 4. 实践项目 C：Feed / 商品列表（带缓存）

在项目 B（笔记）或新开「商品列表」上做均可。

### 4.1 目标行为

1. `GET /v1/notes?limit=20`（或商品列表）走 **Redis → miss 则 DB**。  
2. TTL = 基础值 + 随机抖动。  
3. `PATCH/DELETE` 成功后 **删** 对应用户列表 key。  
4. 不存在的 user/list 可短 TTL 空值，防穿透。  
5. 指标或日志：hit/miss；压测对比关开缓存。

### 4.2 建议接口与 Key

```text
GET  /v1/notes?limit=20&cursor=...
→ key: notes:list:{userId}:{cursor或page版本}

PATCH /v1/notes/{id}
→ DB 更新成功后 DEL notes:list:{userId}:*
  （版本号方案更干净：user 的 list_version++，key 带版本）
```

版本号法（推荐进阶）：缓存 key 含 `list_ver`；写时只让 ver+1，旧 key 自然过期，避免扫 key 删除。

### 4.3 验收清单

- [ ] Compose 增加 Redis 服务  
- [ ] miss 回源路径有单测或集成测  
- [ ] 写后读：可接受短暂旧值或已验证删 key 后必新值  
- [ ] 压测：缓存开启后 DB QPS 下降、P95 下降（记下数字）  
- [ ] Redis 挂掉时的行为明确：报错 or 降级直打 DB（带限流）  
- [ ] 文档写清：一致性窗口与 TTL 策略  

### 4.4 压测对比模板（写进笔记）

```text
场景：20 VU × 30s，同一用户列表接口
无缓存：P95=___  DB QPS≈___  错误率=___
有缓存：P95=___  DB QPS≈___  命中率≈___
结论：...
```

---

## 5. 建议排期（在职 2–3 个月，每周 8–10h）

### 8 周版（紧凑）

| 周 | 主题 | 产出 |
|----|------|------|
| 1–2 | TCP/超时/连接池再实测；Linux 工具箱 | 一张超时图 + 压测观察笔记 |
| 3–4 | 哈希/跳表/堆/树对照 Redis 与索引；刷题启动 | 口述四件套 + 改掉一处 N+1 |
| 5–6 | Redis Cache-Aside + 穿透/击穿/雪崩 | 项目 C 读路径通 |
| 7 | 写后删、TTL 抖动、命中率日志 | 写路径与一致性说明 |
| 8 | 压测对比 + 故障演练（停 Redis） | 出师报告；补面试题 C 组 |

### 12 周版（充裕）

在 8 周上增加：本地二级缓存、list 版本号失效、布隆防穿透、读一篇专栏 064/077 精读笔记、限流小实验（为 L4 铺路）。

### 每周固定节奏

1. **一块系统知识**（网络或 Linux）动手 1 次  
2. **3～5 道题**或 1 个结构对照 Redis 命令  
3. **缓存功能竖切**一点（只做 hit 或只做删 key 也行）  
4. 周末：压测或故障演练，不堆新需求  

---

## 6. 客户端背景：L2 易踩坑

| 坑 | 纠正 |
|----|------|
| 把 Redis 当数据库，无 TTL、无失效 | Cache-Aside + TTL + 写删 |
| 以为有缓存就不用索引 | 先 EXPLAIN，缓存是加成 |
| 全公司共用一个永不过期大 key | 拆 key、防雪崩与热 key |
| 刷题与工作两张皮 | 每周映射到限流/LRU/N+1 |
| 超时到处 30s | 做超时预算表 |
| Redis 挂了雪崩打 DB | 降级与回源限流 |

---

## 7. 专栏与面试对照

| 主题 | 《左耳听风》 | 面试题 |
|------|--------------|--------|
| 理论 / 系统 | 074、075 | H 组部分 |
| Linux / 网络 / 异步 IO | 077、078 | G1 定位慢请求 |
| 缓存 | 064 | **C1–C3** |
| 数据结构直觉 | 074 | 结合 Redis/索引口述 |

正版：[左耳听风](https://time.geekbang.org/column/intro/100002201)

---

## 8. L2 结束后接到哪里？

进入 **L3 软件设计**（分层与范式加固）与 **L4 分布式/弹力**（MQ、熔断限流——缓存之后的下一座山）。

相关文档：

- [01-leveling-guide.md](./01-leveling-guide.md)  
- [05-relational-db-basics.md](./05-relational-db-basics.md)（索引与 EXPLAIN 是缓存的前置）  
- [02-interview-qa.md](./02-interview-qa.md) C 组  

---

## 9. 一页纸检查表

```text
系统与网络
  [ ] 超时预算图
  [ ] 连接池 / FD 有上限意识
  [ ] 会用 ss/top 之一看瓶颈

数据结构
  [ ] 哈希/跳表/堆/树能对应到 Redis 或索引
  [ ] 请求路径无 N+1 / O(n²) 扫库

缓存
  [ ] Cache-Aside 读/写路径正确
  [ ] 穿透、击穿、雪崩各有对策
  [ ] TTL 带抖动；写后删 key
  [ ] 压测对比数据留下了

项目 C
  [ ] Compose 含 Redis
  [ ] hit/miss 可观察
  [ ] Redis 宕机策略明确
```

---

<a id="part-07"></a>

# ========== PART 07：L3 软件设计与范式（07） ==========

# L3 — 软件设计与编程范式详解

> 对应练级文档第 6 章。前置：L1/L2 能交付带库与缓存的接口。  
> L3 **贯穿始终**：不是某一两个月学完就扔，而是每次加功能都用同一套边界与错误模型。  
> 对齐《左耳听风》软件设计、编程范式游记、错误处理；客户端 MVVM/模块化经验大部分可迁移。

---

## 0. 为什么要有 L3？

接口少时，一个 `Service` 里写完也能跑。接口一多会出现：

- Handler 里写 SQL，Repo 里写权限  
- 同一段校验复制四处（改一处漏三处）  
- 异常乱抛，客户端只看到 500  
- 想加限流/日志，只能改每个接口  

L3 要解决的是：**边界清晰、规则可测、错误可预期、横切可插拔**——让系统在变复杂时仍可演进。

### 与客户端同构

| 客户端 | 后端 L3 |
|--------|---------|
| View 不写业务 | Handler 不做业务规则 |
| ViewModel / UseCase | Service / Domain |
| Repository / DataSource | Repo / DAO |
| Interceptor 链式 | Middleware 鉴权/日志/限流 |
| Result / sealed 错误 | 领域错误 vs 系统错误 |

### 出师标准（可持续自检）

- [ ] 新接口默认落在三层，而不是往上帝类里塞  
- [ ] 核心业务规则有单测，不依赖 HTTP  
- [ ] 对外错误码表稳定；对内日志有 request_id + 原因  
- [ ] 鉴权/日志/限流以中间件存在，不复制粘贴  
- [ ] 能说清：这段逻辑为什么放在这一层  

---

## 1. 设计原则（先记会用的）

### 1.1 DRY / KISS / YAGNI

| 原则 | 含义 | 后端落地 |
|------|------|----------|
| **DRY** | Don't Repeat Yourself | 校验、鉴权、错误码集中；别四份拷贝 |
| **KISS** | Keep It Simple | 能用清晰过程式，不上过度抽象 |
| **YAGNI** | You Aren't Gonna Need It | 别预造「未来微服务/通用框架」 |

张力：DRY 过度会变成「牵一发而动全身」的共享泥球——**重复两次可以忍，错误的抽象更贵**。

### 1.2 SOLID（后端常用说法）

| 字母 | 原则 | 后端例子 |
|------|------|----------|
| **S** 单一职责 | 一个模块一件事 | 下单 Service 不负责发邮件（可发领域事件/调 Notifier 接口） |
| **O** 开闭 | 对扩展开放、对修改关闭 | 新支付渠道实现同一 `PaymentGateway`，少改老代码 |
| **L** 里氏替换 | 子类型可替换父类型 | 实现接口的 Mock/真实现行为一致，测才有意义 |
| **I** 接口隔离 | 不强迫依赖用不到的方法 | `UserReader` / `UserWriter` 拆开，避免胖接口 |
| **D** 依赖倒置 | 依赖抽象 | Service 依赖 `NoteRepository` 接口，不依赖具体 SQL 实现 |

客户端映射：协议/接口注入、Repository 抽象——同一套路。

### 1.3 迪米特法则（最少知识）

只跟**直接朋友**说话，别链式钻很深：

```text
# 别这样（Law of Demeter 违规味道）
order.getCustomer().getProfile().getEmail()

# 更好：问直接协作对象
order.customerEmail()
# 或 notification.SendTo(order.CustomerID)
```

后端好处：少耦合内部结构，重构字段时改动面小。

### 1.4 关注点分离：三层（实践默认）

```text
┌─────────────────────────────────────┐
│  Handler / Controller（协议层）      │  HTTP ↔ DTO，鉴权上下文取出
├─────────────────────────────────────┤
│  Service / Domain（领域层）          │  规则、权限、事务边界
├─────────────────────────────────────┤
│  Repository / Infra（基础设施）      │  SQL、Redis、HTTP 客户端
└─────────────────────────────────────┘
```

| 层 | 可以做 | 不要做 |
|----|--------|--------|
| Handler | 解析参数、调 Service、写状态码/JSON | 拼 SQL、开事务细节、复杂 if 业务 |
| Service | 业务规则、事务、调用多个 repo | 依赖 `http.Request` 细节、写响应格式 |
| Repo | CRUD、查询、缓存适配 | 「用户是否有权编辑」这类业务判断 |

进阶（L4+）：六边形 / 整洁架构同一思想——**领域不依赖框架**。L3 先把三层做干净即可。

---

## 2. 编程范式（对齐专栏「编程范式游记」）

不必门门精通；要能在后端**选对工具**。

### 2.1 OOP：领域模型与边界

把「贫血大事务脚本」适度换成有行为的模型：

```text
Note.Rename(title)          # 校验标题规则在模型或 domain service
Notebook.AssertOwnedBy(uid) # 越权检查贴近领域
```

**聚合根直觉（简化）：**  
改笔记时，以 Note（或 Notebook）为入口，保持一组对象一致；别在外面随便改内部子对象破坏不变量。

什么时候别过度 OOP：纯转发 CRUD、无规则的配置读写——过程式 + 校验函数更 KISS。

### 2.2 函数式：纯函数核心逻辑

**纯函数：** 同样输入 → 同样输出，无副作用（不写库、不改全局）。

适合：

- 价格计算、权限判定、状态机迁移是否合法  
- 易单测、易推理  

```text
func CanEdit(actor UserID, note Note) bool
func NextStatus(current, event) (Status, error)
```

副作用（IO）放在外圈 Service 编排里——和客户端「ViewModel 算状态、Repository 做 IO」一致。

### 2.3 泛型：少复制的抽象

用于：

- 通用 `Repository[T]`、分页结构  
- 中间件/装饰器签名  
- Result / Option 一类容器（视语言）  

注意：泛型抽象要有**两个以上真实用例**再抽（YAGNI）。

### 2.4 装饰器 / 中间件：横切关注点

```text
请求 → RequestID → Log → Auth → RateLimit → Handler → …
```

| 横切能力 | 放中间件 | 不放每个业务里复制 |
|----------|----------|-------------------|
| 日志、Trace | ✓ | |
| 鉴权（登录身份） | ✓ | 资源级权限仍常在 Service） |
| 限流、超时 | ✓ | |
| 「这篇笔记能不能改」 | | ✓ Service/领域 |

客户端映射：OkHttp Interceptor 链。

### 2.5 范式怎么选（实用口诀）

```text
结构与生命周期清晰 → OOP / 模块边界
算得清、测得多     → 纯函数
多处相同脚手架     → 泛型（有证据再抽）
所有接口都要的事   → 中间件
```

专栏取向：理解范式背后的**组合与抽象**，而不是追语言新特性名词。

---

## 3. 错误处理（对齐 011 / 012）

错误处理是设计的一部分，不是售后补丁。

### 3.1 两类错误必须分开

| 类型 | 例子 | 对外 | 对内 |
|------|------|------|------|
| **业务错误** | 无权限、不存在、标题冲突 | 4xx + 稳定 code | Warn 日志即可 |
| **系统错误** | DB 挂、未处理 panic、磁盘满 | 5xx + 通用文案 | Error + 堆栈/上下文 |

不要把「用户密码错误」打成 500；不要把「唯一约束冲突」吞掉变成功。

### 3.2 推荐形状（概念）

```text
领域层返回：
  ErrNotFound / ErrPermission / ErrConflict / ErrInvalid
  或 Result{ value | domainError }

Handler 映射：
  domainError → HTTP 状态 + 错误码 JSON
  unknown     → 500 + 日志（不把内部细节回给客户端）
```

**稳定错误码**进契约（OpenAPI），和客户端协商后不要随便改含义。

### 3.3 不要把异常当控制流

- 预期内分支（校验失败、找不到）：用返回值 / 错误类型  
- 真正意外：异常/panic → 中间件 recover → 500  
- 禁止：靠抛异常在正常业务里跳来跳去（难读、难测、易漏 catch）

### 3.4 异步与消息消费（为 L4 铺垫）

消费失败要预设策略：

| 策略 | 何时 |
|------|------|
| 重试（有限次 + 退避） | 暂时性错误（网络抖） |
| 死信队列 | 重试耗尽、毒消息 |
| 幂等 | 至少一次投递下必须 |

同步 API 同样：可重试的客户端操作 ↔ 服务端幂等键（L0/L4 已强调）。

### 3.5 日志与错误一起设计

失败日志至少含：

- `request_id`  
- 操作名（`create_note`）  
- 关键身份（`user_id`，注意脱敏）  
- 错误类型与底层 err  
- 关键输入摘要（勿打密码、Token）  

专栏观点：错误处理策略要统一，异步世界更要预设失败路径。

---

## 4. 实践项目 D：把项目 B/C 重构为清晰分层

目标：**行为不变，结构变干净**——重构不是重写产品。

### 4.1 目标结构（示例）

```text
cmd/server/
internal/
  httpapi/          # handlers + middleware + dto
  service/          # note/notebook/auth 业务
  domain/           # 实体、领域错误、纯函数规则（可选独立）
  repo/             # postgres / redis 实现
  config/
```

### 4.2 重构步骤（建议顺序）

1. **画出当前依赖**：谁 import 了谁；标出 Handler 里的 SQL。  
2. **抽出 Repo 接口**，Service 只依赖接口；先假实现/真实现可切换。  
3. **把权限与校验移进 Service/domain**，Handler 变薄。  
4. **统一错误类型与映射表**；补单测：越权、冲突、校验。  
5. **横切迁到中间件**：RequestID、日志、Auth。  
6. **删除重复代码**；跑集成测与手工回归。  

### 4.3 验收清单

- [ ] Handler 文件中无 SQL、无事务 API  
- [ ] Service 单测不启动 HTTP Server 也能跑核心规则  
- [ ] 错误码表文档化；409/403/404 行为与重构前一致或明确改进  
- [ ] 中间件链可读；新接口加日志零复制  
- [ ] PR 说明：边界图 + 测试证据（不是「顺便大扫除」无重点）  

### 4.4 反模式对照

| 重构前 | 重构后 |
|--------|--------|
| `CreateNote(c *gin.Context)` 里开事务写 SQL | Handler → `svc.CreateNote(ctx, actor, cmd)` |
| 到处 `if err != nil { c.JSON(500,...) }` | 统一 `MapError(err)` |
| 每个函数自己解析 Token | Auth 中间件注入 `ctx.UserID` |
| 业务里 `log.Println(err)` 格式不一 | 结构化字段统一 |

---

## 5. 设计小练习（穿插日常）

每周选一题做 30～60 分钟：

1. 给「改标题」写纯函数校验 + 单测，再接到 Service。  
2. 把发送邮件改成 `Notifier` 接口，默认实现打日志。  
3. 为限流加一个中间件，业务代码零改动。  
4. 审查一个 PR：标出违反分层的三处。  
5. 读专栏一篇范式文，用自己项目里的例子复述。  

---

## 6. 建议节奏（贯穿，而非独占两月）

L3 与 L2/L4 **并行**：

| 节奏 | 做法 |
|------|------|
| 每加一个功能 | 先想放哪一层、错误码是什么 |
| 每两周 | 一次小重构（抽接口/中间件） |
| 每月 | 读 1～2 篇范式/设计专栏，写半页笔记 |
| 项目 D | 在 L2 项目 C 稳定后专门开 1～2 周重构冲刺 |

不要等「学完设计再写代码」——**用设计约束正在写的代码**。

---

## 7. 客户端转后端：L3 易踩坑

| 坑 | 纠正 |
|----|------|
| 把所有逻辑塞 ViewModel 式巨型 Service | 按用例拆 Service 或包 |
| 为未来微服务先拆 20 个库 | YAGNI；先模块边界 |
| 抽象飞起，一层 Wrapper 套一层 | 有两个调用方再抽 |
| 异常当 goto | 返回值表达预期失败 |
| 重构无测试 | 先补特性测试再动结构 |

---

## 8. 专栏与面试对照

| 主题 | 《左耳听风》 | 怎么用 |
|------|--------------|--------|
| 编程范式 | 028–040 | 建立「组合/抽象」语言 |
| 软件设计 | 076 | 原则与设计方法 |
| 错误处理 | 011–012 | 同步/异步错误策略 |
| 面试 | 用项目 D 讲「如何拆分与演进」 | 比背 SOLID 定义更有用 |

正版：[左耳听风](https://time.geekbang.org/column/intro/100002201)

---

## 9. L3 之后

分层与错误模型稳了，再进 **L4 分布式与弹力**：在清晰边界上加 MQ、熔断、限流，而不是在上帝类上缠线。

相关文档：

- [01-leveling-guide.md](./01-leveling-guide.md)  
- [04-l1-entry-and-craft.md](./04-l1-entry-and-craft.md)  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md)  
- [02-interview-qa.md](./02-interview-qa.md)  

---

## 10. 一页纸检查表

```text
原则
  [ ] DRY/KISS/YAGNI 能举例
  [ ] SOLID 能对应到项目一处设计
  [ ] 三层边界说得清

范式
  [ ] 纯函数覆盖至少一块核心规则
  [ ] 中间件承担鉴权/日志等横切
  [ ] 无过度泛型炫技

错误
  [ ] 业务错误 vs 系统错误分离
  [ ] 对外 code 稳定；对内日志完整
  [ ] 无「异常当控制流」

项目 D
  [ ] Handler 无 SQL
  [ ] Service 可单测
  [ ] 回归通过 + PR 有边界说明
```

---

<a id="part-08"></a>

# ========== PART 08：L4 分布式弹力与性能（08） ==========

# L4 — 分布式、弹力与性能详解

> 对应练级文档第 7 章。前置：L1～L3（真库、缓存、清晰分层与错误模型）。  
> 这是客户端转后端的**核心战场**：从「单机做对」到「多节点部分失败时仍可用、可恢复、可观测」。  
> 对齐《左耳听风》：分布式 020–027、弹力 041–051、管理 058–063、性能 064–068、练级 081–085。

---

## 0. L3 → L4：跨的是什么坎？

| | L3 及以前 | L4 |
|--|-----------|-----|
| 失败观 | 异常、回滚事务 | **部分失败**是常态 |
| 调用 | 进程内函数 | 网络调用：超时、重试、重复 |
| 一致 | 单库事务 | 最终一致 + 补偿 |
| 保护 | 连接池、索引 | 限流、熔断、降级、隔离 |
| 交付 | 功能正确 | 功能正确 + **故障演练证据** |

**客户端映射：** 弱网下点两次「下单」——就是分布式里的超时 + 重试 + 缺幂等。你在端上做过的防抖、退避、灰度，在 L4 全部升级成服务端弹力设计。

### 出师标准

- [ ] 能讲清：部分失败、幂等、超时预算、最终一致  
- [ ] 会用至少一套：限流 + 熔断 + 降级（可简化实现）  
- [ ] 会用 MQ 做异步下单，消费侧幂等，能处理重复投递  
- [ ] 有观测：日志 / 基础指标 /（最好）一条链路 trace id  
- [ ] 项目 E 附故障演练记录（kill 下游、打满 QPS、重复消息）  

---

## 1. 分布式难点（先建立问题意识）

专栏「冰与火」与亚马逊实践强调：分布式的难点往往不在「会用某某中间件」，而在**现实约束**。

### 1.1 必须接受的现实

| 难点 | 含义 | 工程后果 |
|------|------|----------|
| **部分失败** | 一次调用里，有的成功有的超时 | 不能假设「全成或全败」；要幂等与对账 |
| **网络不可靠** | 延迟、丢包、分区 | 必须超时；重试会制造重复 |
| **时钟不可靠** | 机器时间可漂 | 慎用本地时钟做唯一顺序；用逻辑或中心发号 |
| **状态难共享** | 多实例内存不一致 | 会话/状态外置；服务尽量无状态 |
| **故障难诊断** | 调用链跨进程 | 要全链路日志/指标/追踪 |
| **发布更危险** | 多版本并存 | 兼容、灰度、回滚预案 |

### 1.2 心智转换公式

```text
单机：   f()  要么抛错要么成功
分布式： f()  成功 | 失败 | 超时（对方可能已成功）
         → 超时后查询或靠幂等重试，不能「再盲调一次当没事」
```

### 1.3 CAP 的工程说法（够用）

分区发生时，你往往在**延迟/可用性**与**强一致**之间权衡。  
实务更多谈：这个写操作能否最终一致？能否用补偿？用户能否接受「处理中」？

---

## 2. 技术栈心智图

先记层次，再选具体产品（Nginx / Kafka / Redis / K8s 都是某一层的实现）。

```text
接入层   DNS / CDN / 网关 / 负载均衡
服务层   无状态服务 + 服务发现 + 配置中心
数据层   主从 / 分片 / 缓存 / 对象存储
异步层   消息队列 / 任务调度 / 工作流
观测层   日志 / 指标 / 链路追踪 / 告警
运维层   容器编排 / 发布策略 / 弹性伸缩
```

| 层 | L4 你要会的事 |
|----|----------------|
| 接入 | 网关限流、TLS、路由；健康检查摘流 |
| 服务 | 无状态水平扩；配置/开关可动态改 |
| 数据 | 缓存 + 主从意识；明白分库分表很贵 |
| 异步 | 用 MQ 削峰解耦；消费幂等 |
| 观测 | 没有指标就没有熔断阈值与复盘 |
| 运维 | 滚动/金丝雀；能回滚 |

**PaaS 本质（专栏方向）：** 把调度、部署、弹性、可观测下沉到平台，应用专注业务——你暂时用 Docker Compose / 单机模拟即可，但心智要对齐。

---

## 3. 弹力设计（Resilience）— 必须会讲会用

对齐 041–051。弹力 = 系统在故障与过载下**弯曲而不折断**，并快速恢复。

### 3.1 模式一览

| 模式 | 一句话 | 落地最小集 | 客户端类比 |
|------|--------|------------|------------|
| **隔离 Bulkhead** | 故障隔舱 | 下游连接池/线程池分开；核心与非核心隔离 | 图片池 ≠ 网络池 |
| **异步通讯** | 削峰解耦 | 非关键路径进 MQ | WorkManager / 本地队列 |
| **幂等** | 一次与多次副作用相同 | 业务唯一键 + 去重表/状态机 | 防重复点击 + 去重 |
| **服务状态** | 无状态优先 | Session 进 Redis；本地内存不当真相 | 别只信进程内单例 |
| **补偿** | 正向失败则反向 | 取消单、释库存、对账任务 | 支付失败回滚 UI+库存 |
| **重试** | 有限次 + 退避 + 抖动 | 只对暂时失败；遵守超时预算 | 指数退避 |
| **熔断** | 下游持续失败则快速失败 | 关闭 → 半开探测 → 关闭 | 连续失败停调 |
| **限流** | 保护自己与下游 | 网关/服务令牌桶或固定窗口 | 防抖 + 服务端配额 |
| **降级** | 牺牲非核心保核心 | 开关关推荐/验真；返回缓存 | 弱网关功能 |

### 3.2 组合口诀（务必背熟）

```text
错误路径：  重试 + 熔断 + 幂等
一致性：    能最终一致就最终一致 + 补偿 / 对账
过载路径：  限流 + 降级
运营底座：  监控 + 健康检查 + 流量调度
```

缺幂等的重试 = 制造双份订单。  
缺熔断的重试 = 雪崩放大器。  
缺观测的限流/熔断 = 拍脑袋阈值。

### 3.3 幂等怎么做（高频）

1. 客户端或网关带 `Idempotency-Key` / 业务订单号  
2. DB **唯一约束**或 Redis `SET NX` 去重  
3. 状态机：`CREATED → PAID`，重复请求返回同一终态结果  
4. 超时后：**先查后改**，不要无脑再 POST 一次  

### 3.4 熔断状态机（口述）

```text
Closed（正常）--失败率超阈值--> Open（快速失败）
Open --冷却时间到--> Half-Open（放少量探测）
Half-Open --成功--> Closed
Half-Open --失败--> Open
```

### 3.5 限流算法直觉

| 算法 | 特点 |
|------|------|
| 固定窗口 | 实现简单，窗口边界可能突刺 |
| 滑动窗口 | 更平滑 |
| 漏桶 | 恒定流出速率 |
| 令牌桶 | 允许一定突发，常用 |

维度：全局限流、按用户、按 IP、按接口——秒杀尤需多层。

### 3.6 降级怎么设计

事前定义：

- 触发条件（错误率、依赖挂、CPU）  
- 牺牲什么（个性化、校验、次要入口）  
- 开关放哪（配置中心 / 环境变量）  
- 演练过没有  

降级不是「挂了再想」，是**产品可接受的简化模式**。

---

## 4. 管理设计（基础设施向）

对齐 058–063。

### 4.1 分布式锁 — 慎用

优先替代：

- DB 唯一约束  
- 状态机 + 条件更新  
- 单分区队列串行消费  

锁的坑：过期业务未完、误删他人锁、当事务用。  
真要用：持有者 token、续期、短临界区。

### 4.2 配置中心

动态：

- 降级开关、限流阈值  
- 功能灰度  
- 非密钥配置  

密钥仍走 Secret；配置变更要可审计、可回滚。

### 4.3 网关 / 边车 / Service Mesh

| 组件 | 作用 |
|------|------|
| **网关** | 南北向入口：鉴权、路由、限流、TLS |
| **边车** | 与应用伴生，接管东西向流量治理 |
| **Mesh** | 平台化的边车体系（Istio 等） |

L4 学习：**先用网关把入口治理做好**；Mesh 知道是什么即可，未必亲自上。

### 4.4 部署升级策略

| 策略 | 特点 |
|------|------|
| 滚动 | 省资源，回滚较慢 |
| 蓝绿 | 两套环境切换，回滚快，成本高 |
| 金丝雀 | 小流量验证再放大 ≈ 客户端灰度 |

配合：健康检查、错误率自动停滚、DB 迁移 expand/contract。

---

## 5. 性能设计

对齐 064–068。L2 已做缓存；L4 把性能放到**架构组合**里。

### 5.1 缓存（复习升级）

- Cache-Aside + TTL 抖动  
- 热 key / 大 key 治理  
- 缓存与 DB 一致性窗口可叙述  

### 5.2 异步处理

能异步的别堵在用户请求里：

```text
同步：下单校验 + 写「预处理订单」 → 快速返回
异步：MQ → 扣库存落库、发短信、积分
```

代价：用户看到「处理中」；要查询接口与补偿。

### 5.3 数据库扩展（知道代价再做）

顺序建议：

```text
索引与 SQL → 缓存 → 读写分离 → 分库分表
```

分库分表代价：跨库事务、Join、扩容迁数、全局 ID。  
**没有证据的分片是事故预告。**

### 5.4 秒杀（极端练手）

套路提纲：

1. 页面静态化 / CDN  
2. 答题或排队削峰  
3. 网关与接口多层限流  
4. Redis 预减库存  
5. MQ 异步建单  
6. DB 唯一约束兜底  
7. 对账与超卖补偿预案  

核心：保护 DB，把压力前移与异步化。

### 5.5 边缘计算（概念）

把能靠近用户的缓存/校验/渲染前移（CDN、边缘节点、端侧）——与客户端缓存、CDN 经验呼应，L4 了解即可。

---

## 6. 观测：弹力的眼睛

没有观测，熔断阈值与复盘都是玄学。

### 6.1 三大支柱

| 支柱 | 内容 | L4 最低配 |
|------|------|-----------|
| 日志 | 结构化 + request_id | 已有则贯穿到 MQ consumer |
| 指标 | QPS、P95、错误率、队列积压、池等待 | Prometheus 或云监控，至少暴露几个数字 |
| 追踪 | 跨服务 span | 至少透传 `trace_id` / `request_id` |

### 6.2 黄金信号（口述）

延迟、流量、错误、饱和度（池/CPU/队列）——定位慢请求按这条链查。

---

## 7. 实践项目 E：简易秒杀 / 抢购

在已有分层服务上做一条「抢购」垂直链路即可，不必仿真整个电商。

### 7.1 范围

1. 活动库存预热到 Redis  
2. `POST /v1/seckill/orders`：限流 + 用户维防刷 + Redis 预减  
3. 发 MQ；消费者建单（订单号唯一 = 幂等）  
4. 模拟支付服务：可配置失败率；超时则查询 + 补偿关单释库存  
5. 支付失败率高时熔断，返回降级提示  

### 7.2 架构草图

```text
Client → Gateway/API(限流)
           → Redis(预减库存)
           → MQ
           → Consumer(幂等建单)
           → Payment(可熔断)
           → DB(唯一订单号)
```

### 7.3 故障演练清单（验收必做）

| 演练 | 操作 | 期望 |
|------|------|------|
| 打满 QPS | k6 抬并发 | 限流生效；核心不全崩 |
| 重复投递 | 同消息消费两次 | 仅一单；第二次幂等成功 |
| Kill 支付 | 支付进程停 | 熔断/快速失败；可补偿关单 |
| Redis 抖动 | 延迟注入或短停 | 有降级或明确错误，不无限重试 |
| 重复点击 | 同 Idempotency-Key | 同单号返回 |

输出一份 **1～2 页演练报告**：现象、指标、改进项。

### 7.4 验收口令

> 我能在部分失败与过载下，靠限流/熔断/幂等/补偿保住系统，并且有演练记录证明。

---

## 8. 建议排期（在职约 2～3 个月）

| 阶段 | 周建议 | 重点 |
|------|--------|------|
| A | 1–2 | 难点认知 + 超时/重试/幂等改造现有写接口 |
| B | 3–4 | 接入 MQ；消费幂等；死信 |
| C | 5–6 | 限流 + 熔断 + 降级开关；基础指标 |
| D | 7–8 | 项目 E 主路径（Redis 预减 + 异步下单） |
| E | 9–10 | 补偿与支付模拟；故障演练文档 |
| F | 11–12 | 补管理/发布阅读；复盘面试题 F/G/E 组 |

可与 L3 小重构穿插：弹力代码也要落在清晰分层里（中间件做限流，领域做幂等键）。

---

## 9. 客户端背景：L4 易踩坑

| 坑 | 纠正 |
|----|------|
| 无脑重试 | 幂等 + 预算 + 熔断 |
| 本地事务里调远程 | 拆本地短事务 + 消息/补偿 |
| 一上来微服务拆很碎 | 先单体模块化 + 异步边界 |
| 分布式锁解决一切 | 优先唯一约束与队列 |
| 只压测不演练失败 | 必须 kill 依赖 |
| 无指标调限流阈值 | 先黄金信号再调参 |

---

## 10. 专栏阅读顺序（后端向）

```text
020–027 分布式全景与监控调度
→ 041–051 弹力设计（可精读幂等/重试/熔断/限流/降级/总结）
→ 058–063 锁/配置/网关/发布
→ 064–068 缓存（复习）/异步/扩展/秒杀
→ 081–085 练级攻略分布式与微服务、容器
```

面试对照：[02-interview-qa.md](./02-interview-qa.md) 的 D/E/F/G 组与秒杀提纲。

正版：[左耳听风](https://time.geekbang.org/column/intro/100002201)

---

## 11. L4 之后 → L5

有了弹力与异步，再谈：**要不要拆微服务、SLO/容量、设计文档驱动演进**（练级文档第 8 章）。

相关文档：

- [01-leveling-guide.md](./01-leveling-guide.md)  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md)（缓存基础）  
- [07-l3-design-and-paradigms.md](./07-l3-design-and-paradigms.md)（分层承载弹力）  

---

## 12. 一页纸检查表

```text
认知
  [ ] 能解释部分失败与超时三态
  [ ] 能画自己系统的层次图

弹力
  [ ] 写路径幂等
  [ ] 重试有界限且不放大雪崩
  [ ] 限流 + 熔断 + 降级至少各一处
  [ ] 补偿或对账路径存在

异步
  [ ] MQ 生产/消费跑通
  [ ] 重复消息安全

性能
  [ ] 缓存策略可叙述
  [ ] 知道分库分表代价

项目 E
  [ ] 秒杀主路径通
  [ ] 故障演练报告写完
```

---

<a id="part-09"></a>

# ========== PART 09：L5 架构进阶与高手成长（09） ==========

# L5 — 架构进阶与「高手成长」详解

> 对应练级文档第 8 章。前置：L4 弹力与异步已有实践与演练。  
> L5 目标：从「能实现」到「能选型、能量化、能讲清演进、能影响协作」——对齐《左耳听风》高手成长篇与领导力/学习/沟通系列。  
> 客户端背景优势：你做过版本兼容、灰度、跨端契约；L5 是把这些升成**服务端架构语言**。

---

## 0. L4 → L5：跨的是什么坎？

| | L4 | L5 |
|--|----|-----|
| 问题 | 这个接口怎么抗住？ | **这个系统该怎么演变成什么样？** |
| 证据 | 压测与故障演练 | SLO、容量粗算、ADR/设计文档 |
| 拆分 | 用 MQ 做异步边界 | 是否/何时拆微服务，数据怎么切 |
| 影响范围 | 自己的服务 | 多团队契约、发布与值班 |
| 成长 | 会模式 | 会权衡 + 会沟通 + 持续学习地图 |

专栏取向：**平凡还是卓越，是你的选择**；高手篇标准高，取法其上。L5 不是头衔，是**可展示的架构产出与影响力**。

### 出师标准（持续滚动）

- [ ] 能独立写出一份完整设计文档（项目 F）并经得起追问  
- [ ] 能判断「现在不该拆微服务」并给出替代（模块化 / 异步）  
- [ ] 能定义至少一组 SLI/SLO，并挂钩告警与降级预案  
- [ ] 能做一页容量粗算（QPS × 成本 × 命中率）  
- [ ] 能主导一次跨端/跨组的接口或故障复盘沟通  

---

## 1. 微服务何时该上

### 1.1 专栏方向（务必先记住）

**先单体模块化，再按边界拆。**  
拆服务的本质是：把**本地函数调用**变成**网络调用**——换来独立扩展与团队自治，换去延迟、部分失败、分布式事务与运维复杂度。

客户端转后端最常见踩坑：业务还不稳就拆成十几个「微服务」，每个都要鉴权、超时、部署、值班 → 复杂度爆炸。

### 1.2 该拆的信号（满足越多越该认真评估）

| 信号 | 说明 |
|------|------|
| 独立扩展 | 某一块 CPU/流量与其它完全不成比例 |
| 独立发布 | 该域变更频繁，却总被无关发布绑死 |
| 团队边界清晰 | 康威定律：组织怎么划，系统容易怎么裂 |
| 数据可拆 | 事务边界清晰，不是强耦合一张大表硬撕 |
| 平台已就绪 | 网关、配置、观测、CI/CD、值班已具备 |

### 1.3 不该拆的信号

- 两人团队养 12 个服务  
- 「以后可能要」的想象拆分（YAGNI）  
- 核心下单路径还在用分布式事务硬刚强一致  
- 没有 trace/指标，拆了无法排障  
- 只是想跟风简历写 Microservice  

### 1.4 更优演进阶梯

```text
V1  模块化单体（包边界清晰，L3 分层）
V2  异步边界（MQ 画出下游，L4）
V3  只把「有证据要独立扩/发」的一块拆成服务
V4  数据迁移与更多服务（谨慎）
```

**模块化单体**已经能解决很多「代码纠缠」问题，却不必付网络税。

### 1.5 拆之前的检查清单（打印用）

- [ ] 画出当前依赖与数据归属  
- [ ] 写清拆分后的失败模式（超时、重复、乱序）  
- [ ] 写清数据拥有者（谁是 Source of Truth）  
- [ ] 同步调用 vs 异步事件的选择理由  
- [ ] 回滚：服务回滚与数据兼容策略  
- [ ] 谁值班、告警算谁的  

---

## 2. 容量与 SLA / SLO

### 2.1 三个词分清

| 词 | 含义 | 例子 |
|----|------|------|
| **SLI** | 怎么度量 | 可用性 = 成功请求 / 总请求；延迟 = P99 |
| **SLO** | 目标值 | 月可用性 99.9%；P99 < 200ms |
| **SLA** | 对外/对合同承诺 | 达不到的后果（赔付等）——常严于内部 SLO |

工程师日常先把 **SLI/SLO** 做扎实；SLA 是产品/商务语言。

### 2.2 建议从最小集合起步

对核心 API 定义：

1. **可用性**（成功比例，排除客户端 4xx 或按约定）  
2. **延迟**（P95 / P99）  
3. **饱和度**（错误率、队列积压、池等待——作前置指标）  

告警对准「快坏 SLO」的信号，而不是 CPU 偶发毛刺。

### 2.3 容量粗算（能上白板即可）

```text
峰值 QPS × 单请求平均成本 ≈ 需要的资源量级

例：
  峰值 2000 QPS
  每请求：1 次 Redis + 0.2 次 DB（命中率 80%）
  → Redis ≈ 2000 QPS；DB ≈ 400 QPS
  再对照单实例能抗的 QPS → 实例数
  留 30%～50% 余量；再叠加限流阈值作安全阀
```

缓存命中率一变，DB 压力可以差一个数量级——**容量叙事里必须写命中率假设**。

### 2.4 预案与 SLO 挂钩

| SLO 风险 | 预案 |
|----------|------|
| 延迟飙升 | 扩容、降级非核心、砍超时下游 |
| 错误率升 | 熔断、回滚、开关 |
| 过载 | 限流、排队、削峰到 MQ |

没有预案的 SLO 只是墙上的数字。

### 2.5 错误预算直觉

若 SLO 是 99.9% 可用，则每月大约允许 **0.1%** 失败时间/请求作「错误预算」——可用来发布与试错；预算耗尽则冻结冒险变更。不必一开始上完整 SRE 体系，但要有这个意识。

---

## 3. 技术领导力与成长（软技能是硬技能）

对齐 005/006、095–099、100–105。后端常做**契约 owner**（对客户端、对下游），不会写设计/不会同步 = 技术债务制造机。

### 3.1 技术领导力（005/006 方向）

不是头衔，而是：

- 用**结果**说话：延迟降了多少、故障少了几次  
- 把复杂问题讲简单，推动决策  
- 抬高团队标准：评审、演练、文档  
- 在关键路径上扛责任（设计、值班、复盘）  

客户端转后端：把你做过的「性能优化复盘」「灰度和崩溃率」改写成服务端叙事。

### 3.2 高效学习（095–099）

| 要点 | 做法 |
|------|------|
| 源头与原理 | 官方文档 / 论文 / 专栏，少追二手名词 |
| 知识地图 | 维护自己的 L0～L5 地图，标薄弱点 |
| 深度与归纳 | 每个主题写半页「自己的话」 |
| 坚持实践 | 设计文档 + 演练 > 只收藏链接 |
| 枯燥与量大 | 切片：本周只攻 SLO 或只攻一个论文 |

专栏态度：没有十年磨一剑的决心，高手路径不适用——对转后端意味着：**持续投入，拒绝速成幻觉**。

### 3.3 高效沟通（100–105）

Talk 与 Code 同等重要：

- 设计评审：先结论与选项，再细节  
- 接口变更：提前通知、兼容策略、时间表  
- 故障：先止损同步，再根因；复盘不甩锅  
- 对客户端：错误码、废弃字段、版本政策书面化  

模板（设计同步一页纸）：

```text
背景 / 目标
方案 A vs B（权衡表）
推荐与理由
影响面（端/下游/数据）
风险与回滚
需要谁决策、何时
```

---

## 4. 架构决策与文档（L5 的日常武器）

### 4.1 ADR（Architecture Decision Record）

短文记录「我们决定了什么、为什么、放弃了什么」：

```text
标题：订单创建改用异步建单
状态：Accepted
上下文：峰值下 DB 连接打满
决策：API 只写预处理单 + 发 MQ
后果：查询「处理中」；需幂等与补偿
备选：同步扩容 3 倍实例（成本高，否决）
```

积少成多，就是团队架构记忆。

### 4.2 C4 / 分层图（够用）

- 系统上下文：谁调用我们  
- 容器：服务、DB、MQ、Redis  
- 组件：模块边界（对应 L3）  
不必追求完美建模工具，**图画对依赖与失败点**就赢过口头架构。

---

## 5. 实践项目 F：设计文档驱动的小型系统

选一个完整、可讲完的场景（二选一或自拟同等复杂度）：

- **短链服务**（创建、跳转、统计、限流）  
- **内容审核流水线**（提交 → MQ → 机审/人审 → 回调）  

### 5.1 文档目录（建议）

1. **背景与目标**（用户是谁、成功长什么样）  
2. **功能需求** + **非功能需求**（QPS、P99、一致性、合规）  
3. **架构图**（上下文 + 容器）与**数据模型**  
4. **关键流程**（时序：创建 / 查询 / 失败）  
5. **失败模式与弹力**（超时、重复、下游挂、过载）  
6. **观测与告警**（SLI/SLO、面板、告警）  
7. **容量粗算**与限流阈值假设  
8. **安全**（鉴权、滥用、隐私）  
9. **演进路线**（V1 单体 → V2 异步 → V3 可选拆分）  
10. **里程碑与风险**  
11. **ADR 附录**（关键决策 2～3 条）  

### 5.2 短链示例：非功能需求样本

```text
创建：P99 < 100ms（不含审核）
跳转：P99 < 50ms；可用性 99.95%
一致性：创建成功后 1s 内跳转可见（可最终一致）
滥用：单用户创建限流；恶意链接异步打标
```

### 5.3 验收

- [ ] 文档他人可读，能按图提出 3 个挑战问题你答得上  
- [ ] V1 可用最小实现跑通（可复用 L0～L4 技术）  
- [ ] 明确写出「现在不拆服务」的理由或「只拆某一块」的理由  
- [ ] SLO + 粗算 + 演练计划写进文档  
- [ ] 做一次 20～30 分钟讲解（练沟通）  

### 5.4 出师口令

> 我能用一份设计文档说明系统如何满足目标、如何失败、如何演进，并推动相关方达成一致。

---

## 6. 高手成长：持续输入清单

对齐练级攻略高手篇取向（系统、数据、分布式、运维、论文），客户端转后端建议优先级：

| 优先级 | 内容 | 产出 |
|--------|------|------|
| P0 | DDIA（数据密集型应用）精读 | 每章半页笔记 |
| P0 | 自己系统的值班/复盘 | 月度复盘一页 |
| P1 | 专栏 027/082 论文索引挑 2～3 篇 | 共识/复制直觉 |
| P1 | 大厂工程博客（缓存/限流/发布） | 对照自家差距 |
| P2 | 容器与编排深入（若岗位需要） | 能看懂部署与探针 |
| P2 | 安全专题（鉴权、注入、密钥） | 威胁建模一页 |

避免：只追新框架名词，不补原理与数据。

---

## 7. 建议节奏（持续 3～6 个月+）

| 节奏 | 做法 |
|------|------|
| 每月 | 一篇 ADR 或设计增量；一次复盘 |
| 每季 | 完整项目 F（或升级上一版设计） |
| 每周 | 学习地图推进 1 格（书/论文/专栏） |
| 并行 | 参与后端值班或 on-call 影子（若有环境） |

L5 没有「学完结业」——是**职业后半段的工作方式**。

---

## 8. 客户端背景：L5 易踩坑

| 坑 | 纠正 |
|----|------|
| 用客户端插件化思维强行拆服务 | 先模块化单体与异步 |
| 设计文档写成功能说明书 | 补非功能、失败、演进、权衡 |
| SLO 抄大厂 99.99 | 按真实依赖与成本定可实现目标 |
| 只做技术不沟通 | 契约变更会议与书面同步 |
| 架构一次画完美 | 小步演进 + ADR |

---

## 9. 专栏对照

| 主题 | 《左耳听风》 |
|------|--------------|
| 分布式与微服务 | 020–027、084、081–083 |
| 容器运维 | 085 |
| 领导力 | 005、006、010 |
| 学习 | 095–099 |
| 沟通 | 100–105 |
| 面试与实力 | 091–094 |
| 练级高手篇总方向 | 069、090 及高手成长各章 |

正版：[左耳听风](https://time.geekbang.org/column/intro/100002201)

---

## 10. 与全书衔接

```text
L0 能提供 API
L1 能上预发（数据与规范）
L2 能变快（系统 + 缓存）
L3 能演进代码结构
L4 能抗住部分失败
L5 能决定形态、量化目标、带人把事讲清楚
```

相关文档：

- [01-leveling-guide.md](./01-leveling-guide.md)  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md)  
- [07-l3-design-and-paradigms.md](./07-l3-design-and-paradigms.md)  
- [02-interview-qa.md](./02-interview-qa.md) J 组与场景题  

---

## 11. 一页纸检查表

```text
微服务
  [ ] 能说出拆与不拆的证据
  [ ] 演进阶梯写进项目 F

SLO / 容量
  [ ] 核心 API 有 SLI/SLO
  [ ] 有一页粗算与预案

领导力与沟通
  [ ] 有 ADR 或设计评审记录
  [ ] 能讲清一次权衡决策

项目 F
  [ ] 文档十章要素齐全（可裁剪）
  [ ] 讲解过；挑战问题答得上
  [ ] V1 跑通或明确范围裁剪
```

---

<a id="part-10"></a>

# ========== PART 10：日志冷热 HDFS + ES + Nginx（10） ==========

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

### 2.2.1 Bulk 瘦文档是怎么「起作用」的

两件事叠在一起：**瘦** = 每条 payload 小；**Bulk** = 很多条一次发给 ES。

```text
HDFS 上报包内一行（胖）
  eventTime, module, file, line, userId, extra(可能很大)
        │
        ▼ toThinDoc
ES 瘦文档（小）
  eventTime, module, file, line, userId, extraPreview(截断)
  —— 丢掉完整 extra ——
        │
        ▼ 凑满 500～2000 条
ES Bulk API 一次提交
  { index: {_index, _id: rowKey} }
  { ...thin... }
  { index: ... }
  { ...thin... }
  …
```

| 环节 | 起什么作用 |
|------|------------|
| **丢掉 extra** | 索引体积、倒排、`_source`、merge、堆内存都小；列表本来也不展示全文 |
| **只留主维+preview** | 之后 `filter(userId+时间+module)` + 分页能直接在 ES 完成，**不必再拉 HDFS 包** |
| **Bulk 一批写** | 比逐条 `index` 少几十～上百倍 HTTP/刷新开销；回填吞吐上去 |
| **`_id=rowKey` 幂等** | 同一行灌两次是覆盖/冲突，不产生重复文档；惊群重试安全 |
| **异步 Bulk + 首屏直出** | 用户先拿到内存筛好的一页；ES 在后台变热，**第二次查询走 ES 才「感受到」瘦文档的收益** |

对查询路径的效果（第一次 vs 第二次）：

```text
冷（无瘦文档）：userId → HDFS 最新包 → 内存过滤 → 返回
                └─ 同时 Bulk 瘦文档进 ES，Redis 打 warm

热（有瘦文档）：list 条件 → ES search_after → 返回
                └─ 不再读 HDFS（除非展开全文再取包）
```

一句话：**瘦文档让「列表要的字段」廉价地住在 ES 里；Bulk 让这次搬家搬得快；warm 之后列表才稳定享受 ES 速度。**

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
> **ES**：列表 miss 时从该包过滤后 Bulk 瘦文档；**warm/锁在 Redis（含 uploadId）**。  
> **全文**：再取同包，用行字段匹配；不写 ES。

---

## 12. 相关文档

- [02-interview-qa.md](./02-interview-qa.md) — I4  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md)  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md)

---

<a id="part-02"></a>

# ========== PART 02：面试题与参考答案（02） ==========

# 后端开发面试题与参考答案

> 面向：**客户端背景转后端 / 全栈** 候选人。  
> 题目按主题分层；每题含「考察点」「参考答案」「客户端迁移提示」。  
> 体系呼应《左耳听风》分布式、弹力设计、性能设计与面试攻略——**实力是王中王，答案要能落到你做过的项目**。

---

## 使用建议

1. 先闭卷口述 2 分钟，再对照参考答案补洞。  
2. 每个主题至少准备 1 个「自己项目里的故事」。  
3. 面试官追问时，优先讲权衡（trade-off），少背名词。

---

## A. HTTP / API / 鉴权

### A1. GET 和 POST 的区别？幂等是什么？

**考察点**：HTTP 语义、幂等、安全方法。

**参考答案**：  
- **安全（Safe）**：不修改资源语义上的状态，如 GET/HEAD。  
- **幂等（Idempotent）**：同一请求执行一次与多次，对资源的副作用相同。GET/PUT/DELETE 幂等；POST 默认不幂等。  
- 实际工程中：浏览器缓存、CDN、重试、网关都会假设语义正确；把「创建订单」做成可随意重试的 POST 而不带幂等键，会导致重复下单。

**客户端迁移提示**：客户端弱网重试 ≈ 服务端必须幂等；你曾用「防重复点击」，后端要用 `Idempotency-Key` / 唯一业务单号。

---

### A2. 如何设计一个不会破坏老客户端的 API 演进？

**考察点**：兼容性、版本策略。

**参考答案**：  
1. **加字段兼容，删字段谨慎**：只增可选字段；废弃字段保留至少一个大版本。  
2. **版本策略**：URL `/v1`、Header `Accept-Version`，或仅靠兼容演进（推荐内部优先兼容）。  
3. **枚举要可扩展**：未知枚举值客户端要能降级显示。  
4. **契约测试**：Provider/Consumer 测试防止偷偷改语义。  
5. **错误码稳定**：业务错误码不要频繁复用改含义。

**客户端迁移提示**：你最懂「后端改字段导致崩溃」——把这份痛苦写进你的设计原则。

---

### A3. JWT 与 Session 怎么选？Token 泄漏怎么办？

**考察点**：无状态鉴权、安全。

**参考答案**：  
- **Session**：服务端存会话，易注销、易控并发登录；需粘滞或集中 Session 存储。  
- **JWT**：校验可本地化，利扩展；难强制下线，payload 勿放敏感信息，要短过期 + Refresh Token，可结合黑名单/版本号作废。  
- **泄漏应对**：短 TTL、刷新轮转、设备绑定、异常登录检测、HTTPS-only、避免放 LocalStorage（Web）等。

---

### A4. 接口超时在客户端设 10s，服务端应如何配合？

**考察点**：超时、重试、级联失败。

**参考答案**：  
超时必须**端到端预算**：网关 3s、服务 2s、下游 1s，层层更短，避免客户端已放弃服务端还在算。  
配合：明确幂等、可取消（若协议支持）、慢查询治理、熔断，禁止无脑长超时。

---

## B. 数据库与事务

### B1. 事务的隔离级别有哪些？默认常用哪个？

**考察点**：ACID、并发异常。

**参考答案**：  
- 读未提交 / 读已提交 / 可重复读 / 串行化。  
- MySQL InnoDB 默认**可重复读**；PostgreSQL 默认**读已提交**。  
- 异常：脏读、不可重复读、幻读；用锁或 MVCC 缓解。  
- 业务上：不是级别越高越好，级别越高并发越差；关键写路径用约束 + 正确事务边界。

---

### B2. 什么时候建索引？索引一定会变快吗？

**考察点**：B+Tree、选择性、写放大。

**参考答案**：  
- 高频等值/范围查询、排序、Join 字段考虑索引。  
- 高区分度列更适合；低区分度（如性别）单独索引收益差。  
- 联合索引遵循最左前缀；注意覆盖索引减少回表。  
- 代价：写入变慢、占空间；过多索引拖累写入。用 `EXPLAIN` 验证。

---

### B3. 如何避免超卖（库存扣减）？

**考察点**：并发写、约束、CAS。

**参考答案**（由易到难）：  
1. DB：`UPDATE stock SET n=n-1 WHERE id=? AND n>=1`，检查 affected rows。  
2. 唯一约束 / 乐观锁 version 字段。  
3. Redis 预减 + 异步落库（秒杀）；以 DB 为准做对账与补偿。  
4. 悲观锁 `SELECT FOR UPDATE`（吞吐差，适合低并发强一致）。

**客户端迁移提示**：类似「本地库存乐观更新失败再回滚 UI」，但后端必须以约束为真相。

---

### B4. 分库分表什么时候做？有什么代价？

**考察点**：扩展、复杂度。

**参考答案**：  
- **先优化**：索引、缓存、读写分离、归档。单表过大、写入瓶颈、运维窗口不够再拆。  
- **代价**：跨库事务难、Join 难、扩容迁数难、全局有序 ID、二刷查询复杂。  
- 拆分键选择决定命运（用户维 vs 订单维）。

---

### B5. MySQL 主从延迟，读到旧数据怎么办？

**考察点**：读写分离一致性。

**参考答案**：  
- 关键读走主；或会话级「写后读主」一段时间。  
- 缓存作为读源时注意失效策略。  
- 业务接受最终一致则提示「处理中」。  
- 监控 Seconds_Behind_Source，延迟过大摘除从库流量。

---

## C. 缓存

### C1. 缓存穿透、击穿、雪崩分别是什么？如何防？

**参考答案**：  

| 问题 | 含义 | 防护 |
|------|------|------|
| 穿透 | 查不存在的数据，打穿到 DB | 布隆过滤、空值短缓存、参数校验 |
| 击穿 | 热点 key 过期瞬间大量回源 | 互斥重建、逻辑过期、永不过期+异步刷新 |
| 雪崩 | 大量 key 同时过期或 Redis 挂 | TTL 加随机、多级缓存、熔断降级、集群高可用 |

---

### C2. 先更新数据库还是先更新缓存？

**考察点**：Cache-Aside。

**参考答案**：  
常用 **Cache-Aside**：写 DB 成功后**删除**缓存（而非更新缓存）。  
原因：更新缓存易产生并发下的脏值；删缓存让下次读回源。  
配合：延迟双删、订阅 binlog 删缓存等进阶方案按一致性要求选型。

---

### C3. Redis 为什么快？单线程还快吗？

**参考答案**：  
内存操作、高效数据结构、I/O 多路复用；命令执行单线程避免锁竞争（现代版本网络等模块有 I/O 线程）。  
快不等于无瓶颈：大 key、热 key、慢命令（Keys）、持久化与 fork 仍可拖垮。

---

## D. 消息队列与异步

### D1. 为什么要用 MQ？不要用 MQ 的场景？

**参考答案**：  
**适合**：削峰、解耦、异步化（通知、审核、账单）。  
**不适合**：强同步强一致短链路、团队无消费治理能力、消息顺序与事务要求极高却设计混乱。  
MQ 引入：重复消费、乱序、积压、运维复杂度。

---

### D2. 如何保证消息「至少一次」下的业务正确？

**考察点**：投递语义 + 幂等。

**参考答案**：  
多数 MQ 是 **At-least-once**。业务侧必须幂等：  
- 唯一业务键 + 去重表  
- 状态机：只允许合法迁移（CREATED→PAID）  
- 消费成功再 ack；失败重试进死信  
Exactly-once 往往是「至少一次 + 幂等」的工程近似。

---

### D3. 消息积压怎么处理？

**参考答案**：  
扩容消费者、临时提高分区/并行度、降级非核心消费、紧急丢弃或采样（需业务允许）、先止血再查生产突增原因。  
预防：监控 lag、限流生产、合理分区键。

---

## E. 分布式与一致性

### E1. 用自己的话解释 CAP / 基础可用性与最终一致

**参考答案**：  
分区发生时，系统在 **线性一致（C）** 与 **可用性（A）** 间权衡。  
实务更多谈：**延迟、吞吐、一致性模型、故障域**。电商详情可最终一致；支付扣款要强约束。  
「BASE」强调：基本可用、软状态、最终一致——用补偿与对账补齐。

---

### E2. 分布式事务有哪些方案？怎么选？

**参考答案**：  
- **本地事务 + 可靠消息 / 出箱模式（Outbox）**：最常用  
- **Saga / TCC / 补偿**：长事务、跨服务  
- **2PC**：强一致但性能与可用性差，少用  
选型：优先避免分布式事务——把边界收进单服务单库；必须跨服务则最终一致 + 补偿。

---

### E3. 分布式 ID 怎么生成？

**参考答案**：  
UUID（无序）、DB 号段、Snowflake 类（时间+机器+序列）、Leaf 等。  
关注：趋势有序（利索引）、时钟回拨、机器号分配、溢出。

---

### E4. 分布式锁怎么用？什么时候不该用？

**参考答案**：  
Redis（SET NX EX + 持有者校验 + 续期）、ZK/etcd。  
**陷阱**：锁过期业务未完、误删他人锁、红锁争议、当事务用。  
**更好替代**：DB 唯一约束、队列串行、状态机。  
专栏取向：管理设计里「锁」是工具，不是第一选择。

---

## F. 弹力设计（高频，对齐专栏 041–051）

### F1. 什么是熔断？和限流、降级的区别？

**参考答案**：  

| 手段 | 目的 | 典型行为 |
|------|------|----------|
| 限流 | 保护系统不被过量流量打垮 | 拒绝/排队/等待令牌 |
| 熔断 | 下游持续失败时快速失败，防止雪崩 | 打开→拒绝→半开探测 |
| 降级 | 资源不足时保核心体验 | 关推荐、返回缓存、简化流程 |

三者常组合：入口限流，下游熔断，极端降级。

---

### F2. 重试有什么危害？如何安全重试？

**参考答案**：  
危害：放大流量、重复副作用、重试风暴。  
安全重试：只对**幂等且可能暂时失败**的操作；有限次数；指数退避 + 抖动；有超时预算；熔断后不再重试；区分 4xx/5xx。

---

### F3. 讲一下幂等性的实现方式

**参考答案**：  
1. 全局唯一请求 ID / 业务单号  
2. 去重存储（Redis SETNX 或 DB 唯一索引）  
3. 状态机保证重复请求返回同一结果  
4. HTTP：PUT/DELETE 语义；POST 配 Idempotency-Key  
5. 支付等场景：先查询再下单，或对账补偿

---

### F4. 隔离设计（舱壁）如何落地？

**参考答案**：  
线程池/连接池按依赖隔离；按租户或业务分片；核心与非核心资源隔离；容器/集群故障域隔离。  
一个慢 SQL 不应占满所有 Tomcat 线程——这与客户端「图片池与 API 池分离」同构。

---

### F5. 补偿事务是什么？举业务例子

**参考答案**：  
正向操作失败或超时后，用**可回撤的反向操作**恢复近似一致。  
例：订票成功但支付失败 → 释放座位；发券失败 → 重试或作废订单权益。  
补偿要幂等、可观测、可人工介入。

---

## G. 性能、网关与部署

### G1. 如何定位一次线上接口变慢？

**参考答案**：  
1. 指标：P99、QPS、错误率、饱和度（线程/连接/CPU）  
2. 链路追踪：慢在哪一跳  
3. 日志：请求 ID 串联  
4. DB：慢查询、锁等待  
5. 依赖：下游 RT、缓存命中率  
6. 变更：是否刚发布/配置变更  

对齐专栏「全栈监控」：没有观测等于盲修。

---

### G2. 网关的职责是什么？

**参考答案**：  
统一鉴权、路由、限流、TLS、灰度、协议转换、访问日志。  
业务逻辑尽量不放网关；网关是流量与策略层（管理设计「网关模式」）。

---

### G3. 滚动发布、蓝绿、金丝雀区别？

**参考答案**：  
- **滚动**：逐步替换实例，省资源，回滚较慢。  
- **蓝绿**：两套环境切换，回滚快，成本高。  
- **金丝雀**：小流量验证再放大，与客户端灰度同源。  

配合健康检查、自动回滚指标。

---

### G4. 秒杀系统怎么设计（提纲）？

**参考答案**：  
前端答疑 + 按钮防重；CDN 静态化；网关限流；答题/排队；Redis 预减库存；MQ 异步下单；幂等落单；库存对账；降级预案。  
核心思想：把压力前移与异步化，保护 DB（性能设计「秒杀」）。

---

## H. 操作系统与语言（客户端转后端常问）

### H1. 进程与线程区别？协程呢？

**参考答案**：  
进程：独立地址空间；线程：共享内存，调度开销更小；协程：用户态轻量并发，适合高并发 IO（Go goroutine、Kotlin 协程）。  
后端要关心：阻塞点、池化、CPU 密集与 IO 密集分离。

---

### H2. 说说连接池为什么重要？

**参考答案**：  
创建连接贵（握手、认证、内存）。池化复用控上限，防止打爆 DB。  
参数：最大连接、空闲、超时、生命周期；与 DB `max_connections`、实例数相乘要算总账。

---

### H3. Java：谈谈 GC 或 Spring Bean 生命周期（若面 Java）

**参考答案提纲**：  
分代假说、Minor/Major GC、常见收集器差异；Bean 作用域、循环依赖、注入时机。  
（按岗位深度展开；客户端转 Java 要诚实说实践深度。）

### H4. Go：GMP 是什么？（若面 Go）

**参考答案提纲**：  
Goroutine / Machine / Processor 调度模型；信道与锁的选择；不要为了「并发」泄露 goroutine。

---

## I. 场景设计题（综合）

### I1. 设计短网址服务

**要点**：  
- `POST` 创建：长链 → 短码（哈希取模 / 发号）；防冲突  
- `GET` 302/301 跳转；缓存热点；统计异步  
- 限流防刷；恶意链接审核异步化  
- 数据：短码唯一索引；过期策略  

### I2. 设计用户 Feed 流

**要点**：  
推模式 / 拉模式 / 推拉结合；缓存时间线；扇出代价；删帖与隐私；分页 cursor；降级只出关注人未读。

### I3. 客户端上报日志 / 埋点接入

**要点**：  
网关鉴权与限流；批量写入；Kafka 缓冲；乱序与重复；采样；隐私脱敏；回压策略。  
（客户端出身可强调端上批量与服务端契约。）

### I4. 日志查询：时间+功能模块索引是否够用 / 列表与全文（Node.js）

**要点**：  
- 列：`eventTime`、`module`（**高重复**）、`file`/`line`、`extra`（**几乎不重复**）  
- **列表 query**：**userId + from/to + module + 可选筛选**；**无 logId、无 fileUri**  
- **HDFS 取数**：**userId → 该用户最近一次上报的日志包**；包内再按时间/模块过滤  
- **ES**：仅列表 miss → 解析该包 → Bulk 瘦文档；换 module 复用同一包侧写  
- **全文**：再取同包，用行字段匹配；不写 ES；不做 logId/fileUri 回源  
- 见 [10-log-cold-hot-es-nginx.md](./10-log-cold-hot-es-nginx.md) §1 / §3 / §5  

完整方案见该文档。

### I5. 本项目相关的 Node.js 面试题（日志冷热查询）

> 场景锚点：Node 做 Log Query API；HDFS=`userId→最近一次上报`；ES=列表瘦索引；Redis=warm/锁。

#### N1. 为什么用 Node 做这个查询服务？有什么风险？

**参考答**：  
适合 IO 密集（等 HDFS / ES / Redis）。风险：单线程 event loop，JSON 解析大包、同步压缩会堵全进程；多实例下内存 Map 锁不够。  
对策：流式解析、Worker 卸 CPU、超时、连接池、Redis 分布式锁。

#### N2. 同一 userId 多人同时查冷数据，Node 怎么防惊群？（含实现）

**参考答（口述）**：  
进程内 `Map` singleflight（同 `userId+uploadId` 共用 Promise）+ Redis `SET NX` 回填锁；首屏可共享；Bulk 异步。只靠进程内 Map 在多 Pod 下无效。

**实现要点（两层合流）**：

```text
层1 进程内：inflight Map —— 同 key 的 await 都挂在同一个 Promise 上
层2 跨进程：Redis SET key NX EX —— 只有一个 Pod 真去拉 HDFS/Bulk
```

```js
// --- singleflight：多人共用一次执行结果 ---
const inflight = new Map();

function singleflight(key, fn) {
  const hit = inflight.get(key);
  if (hit) return hit;                 // B、C 直接拿 A 的 Promise

  const p = (async () => {
    try {
      return await fn();               // 只有第一个进来的跑 fn
    } finally {
      inflight.delete(key);            // 必须删，否则泄漏 / 永远复用旧 Promise
    }
  })();

  inflight.set(key, p);
  return p;
}

// --- 冷回填：同 userId+uploadId 只拉包一次 ---
async function ensureBackfill(userId, uploadId, doWork) {
  const key = `bf:${userId}:${uploadId}`;

  return singleflight(key, async () => {
    // 跨 Pod：只有一个实例拿到锁
    const locked = await redis.set(`backfill:${userId}:${uploadId}`, '1', 'EX', 120, 'NX');
    if (!locked) {
      // 别人在灌：轮询 warm，或短等后再读 ES / 共享方结果
      await waitUntil(() => redis.get(`warm:${userId}:${uploadId}`), { timeoutMs: 3000 });
      return { shared: true };
    }
    try {
      const page = await doWork();     // getLatestUpload → 过滤 → 首屏 + 异步 Bulk
      await redis.set(`warm:${userId}:${uploadId}`, '1', 'EX', 86400);
      return { page, shared: false };
    } finally {
      await redis.del(`backfill:${userId}:${uploadId}`);
    }
  });
}
```

并发回数据怎么走（冷）：

```text
T0  A、B、C 同时 miss（同 userId，同 uploadId）
T1  A 创建 Promise，放进 inflight；B、C get 到同一个 Promise
T2  A 的 fn：Redis SET NX 成功 → 拉 HDFS → 凑满 page
T3  A resolve({ page })  → B、C 的 await 一起拿到同一份 page（或自行再 es.search）
T4  Bulk 可继续异步；warm 点亮后后续请求走 ES
```

**相同 listKey 只打一次 ES search，多人怎么拿数据**（热路径，同一套路）：

```js
async function searchOnce(listKey, runSearch) {
  // 1) 短缓存：已有结果直接返回
  const cached = await redis.get(`list:${listKey}`);
  if (cached) return JSON.parse(cached);

  // 2) singleflight：并发只 search 一次
  return singleflight(`es:${listKey}`, async () => {
    // 双检，避免等锁期间别人已写入缓存
    const again = await redis.get(`list:${listKey}`);
    if (again) return JSON.parse(again);

    const result = await runSearch();          // 真正的 es.search
    await redis.setex(`list:${listKey}`, 15, JSON.stringify(result));
    return result;                             // 所有 await 方拿到同一 result
  });
}

// 三个并发请求：
// const r1 = searchOnce(key, () => es.search(...))
// const r2 = searchOnce(key, () => es.search(...))
// const r3 = searchOnce(key, () => es.search(...))
// → 只调用 1 次 es.search；r1/r2/r3 都 resolve 为同一对象（或同等 JSON）
```

要点：

| 点 | 说明 |
|----|------|
| **回传机制** | 不是「推送」，是多个 HTTP handler `await` 同一个 Promise，`resolve` 后各自 `res.json` |
| **listKey** | 必须含 userId、from、to、module、辅筛、cursor，防串条件/串租户 |
| **多 Pod** | 仅 Map 时每 Pod 仍可能各打一次 ES → 再加 Redis 锁或依赖短缓存「第二个 Pod 读到 cache」 |
| **权限** | 缓存与 flight key 都要带 userId |

#### N3. Bulk 瘦文档是什么？为什么不把完整 extra 进 ES？

**参考答**：  
`toThinDoc` 去掉完整 `extra`，只留时间/模块/file/line/preview/userId；Bulk 一批写入。  
extra 基数高、体积大，进 ES 会撑爆倒排与 `_source`；列表不需要全文，全文再读 HDFS 包匹配。  
**作用**：冷→热后列表走 ES filter，不再每次拉包。

#### N4. 首屏如何快速返回？和 Bulk 的关系？

**参考答**：  
拉最新上报包 → 流式过滤 → **凑满一页先 `return`**；命中行 **异步 enqueueBulk**，不 await 整包进 ES。  
第二次请求看 Redis `warm:{userId}:{uploadId}`，走 ES。

#### N5. warm / 回填状态为什么放 Redis 不放内存？

**参考答**：  
多实例要共享「是否已灌、谁在灌」；进程重启不能丢锁导致惊群。key 必须带 `uploadId`，新上报后旧 warm 失效。

#### N6. 用户先查 network 再查 route，Node 侧怎么优化？

**参考答**：  
同一 `getLatestUpload(userId)` 只拉一次；解析时侧写包内所有 module 瘦文档；当前 module 优先凑页。listKey 含 module，结果缓存不串。

#### N7. event loop 在本场景最容易被什么堵死？

**参考答**：  
整包同步 `JSON.parse`、大字符串正则脱敏、同步读文件、超大 `JSON.stringify` 响应。  
改为流式按行 parse、限制 preview 长度、全文单独接口、监控 event loop lag。

#### N8. 如何设计超时预算（对齐 Nginx）？

**参考答**：  
例：Nginx `proxy_read_timeout` 5s → API 总预算 4s → ES ≤200ms，HDFS 流式首屏尽量短，Bulk 不计入用户路径。客户端断开后回填可改队列异步，避免无限挂起。

#### N9. ES / HDFS 客户端连接要注意什么？

**参考答**：  
共用带 `keepAlive` 的 Agent；设 `maxSockets`；禁止每请求 `new Client()`；ES 客户端限制重试防故障放大；计入连接池总账。

#### N10. 展开全文时 Node 做什么、不做什么？

**参考答**：  
再取（或缓存）该 user 最新包，用列表行字段匹配返回 `extra`。  
**不** `es.index` 全文、**不**用 logId/fileUri。同包展开可 singleflight，并按 user 限流。

#### N11. 如何观测这套 Node 服务？

**参考答**：  
`event_loop_lag`、heap、HDFS/ES RT与错误率、`backfill_inflight`、warm 命中率、Bulk 429、列表 P99；日志带 `request_id`（透传 Nginx）。

#### N12. 手写一段 singleflight + Redis 锁的思路（口述即可）

**参考答**：  
`inflight.get(key) ?? 创建 Promise`；Promise 内 `SET key NX EX`；拿到锁则拉包/Bulk/SET warm；否则 sleep 轮询 warm；`finally` 删除 inflight 与锁。key=`userId:uploadId`。

---

## J. 行为与学习方法（对齐专栏面试/学习篇）

### J1. 为什么从客户端转后端？

**参考答法**：  
结合真实动机：想补齐全链路、对数据与稳定性感兴趣、业务需要全栈。  
展示已做准备：项目、专栏学习路径、能讲弹力设计。避免「客户端没前途」这类否定式表述。

### J2. 遇到线上故障你怎么做？

**参考答法（对齐故障处理最佳实践）**：  
止损（回滚/限流/降级）→ 定位（观测）→ 恢复 → 复盘（根因、改进、演练）→ 沉淀机制。  
强调：**先恢复业务，再追求完美根因**。

### J3. 最近深入读过什么技术内容？

**参考答法**：  
点名《左耳听风》某一章（如幂等或熔断），用自己的话 + 项目实践复述，忌「看过很多」却讲不出细节。

---

## K. 快速自测清单（30 题口令版）

能在 30 秒内说清要点即可过关：

1. 幂等定义  
2. 事务隔离级别  
3. 索引最左前缀  
4. 缓存三大问题  
5. Cache-Aside  
6. 至少一次 vs 至多一次  
7. 消费者幂等  
8. CAP 的工程含义  
9. Saga/补偿  
10. 分布式锁替代方案  
11. 熔断状态机  
12. 令牌桶 vs 漏桶  
13. 降级例子  
14. 重试抖动  
15. 超时预算  
16. 连接池总账  
17. 主从延迟读  
18. 超卖 SQL  
19. 分库分表代价  
20. 网关职责  
21. 金丝雀发布  
22. P99 是什么  
23. 链路追踪用途  
24. 雪崩防护  
25. Outbox 模式  
26. Snowflake 组成  
27. 空值缓存  
28. 死信队列  
29. 读写分离场景  
30. SLO vs SLA  

不会的题 → 回到 [01-leveling-guide.md](./01-leveling-guide.md) 对应级别补齐。

---

## L. 参考答案评分标准（面试官视角）

| 档位 | 表现 |
|------|------|
| 及格 | 能定义概念，有基本方案 |
| 良好 | 能讲权衡、失败模式、观测 |
| 优秀 | 能结合流量数字、演练经验、演进路线 |
| 杰出 | 能指出方案边界，并提出更简替代（如用唯一约束代替分布式锁） |

《左耳听风》面试篇的核心：**准备项目故事、展示思考深度、实力重于话术**。客户端背景是加分项——当你能把端上弱网、重试、灰度经验迁移到弹力设计语言时，往往比「纯背题后端」更有辨识度。

---

## 附录：题目与专栏章节对照

| 主题 | 建议精读专栏方向 |
|------|------------------|
| 错误与重试 | 011–012、047 |
| 分布式全景 | 020–027、081–083 |
| 弹力设计 | 041–051 |
| 锁/配置/网关/发布 | 058–063 |
| 缓存/异步/扩展/秒杀 | 064–068 |
| 微服务与容器 | 084–085 |
| 面试与学习 | 091–099 |

---

# 附录：合并说明

- 本文为拆分笔记的合并副本，便于复制到飞书/Notion/本地。
- 若与分册冲突，以各分册最新内容为准；可重新运行合并脚本生成。
- PART 顺序：01 → 11/12（L0 加深）→ 03～09 → 10（日志专项）→ 02（面试）。
