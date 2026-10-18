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
