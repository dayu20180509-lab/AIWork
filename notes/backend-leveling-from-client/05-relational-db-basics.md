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
