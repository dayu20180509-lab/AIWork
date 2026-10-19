# 日志冷热架构：大文件 + ES 回填 + Nginx 限流

> 场景：日志已写入**大文件系统（只写追加）**；前端按 **logId** 调接口；后端先查 **ES**，没有则回源文件系统再**回填 ES**。  
> 本文合并：读路径设计、并发写、ES 查询慢、ES 过期，以及 **Nginx 多用户查询 / 同 logId 查询** 怎么配、和后端如何配合。

---

## 1. 架构定位

| 层 | 角色 |
|----|------|
| 大文件系统 | **真相源**，只写（追加），按 logId 可读 |
| Elasticsearch | **热层 / 检索加速**，可丢可重建，带过期 |
| API 服务 | Cache-Aside：先 ES → miss 回源 → 回填 |
| Nginx | 入口 TLS、路由、**限流与连接保护** |

```text
多用户浏览器
    → Nginx（限流 / 连接 / 路由）
    → Log Query API
         → ES.get(logId)
              hit  → 返回
              miss → 文件服务.get(logId) → 写 ES → 返回
```

原则：ES 挂了仍应能回源返回；回填失败不能吞掉文件内容。

---

## 2. 读路径：接口要做的事

```text
GET /api/logs/{logId}

1. 鉴权授权（谁能看这条日志）
2. 参数校验（logId 格式）
3. ES 点查（优先 GET _doc/{logId}，可加 routing）
4. hit 且未逻辑过期 → 返回
5. miss →
     a. 互斥 / singleflight（见第 3 节）
     b. 调大文件系统读原文（超时预算）
     c. 不存在 → 404（可短负缓存防穿透）
     d. 规范化、脱敏、裁剪 _source
     e. 幂等写入 ES（_id=logId）
6. 返回正文；打点 hit/miss、回源 RT、ES 写失败
```

可选：

- 先返回文件内容，**异步**回填 ES（降首包延迟）  
- 极热 logId 再加一层 Redis 短缓存  

---

## 3. 并发：多请求同时查同一个 logId

### 3.1 问题

同一 `logId` 在 ES 未命中时，多个用户/标签页同时请求 → 集体 miss → 重复打文件服务 → 并发写 ES。

### 3.2 后端解决方案（主方案）

| 手段 | 做法 |
|------|------|
| **Singleflight / 互斥** | 同一 `logId` 只允许一个回源；其它等待或复用结果 |
| **分布式锁** | 多实例时用 `SET NX EX backfill:{logId}`，拿到锁的才回源写 ES |
| **幂等写 ES** | `_id = logId`；`index` 覆盖或 `create` 遇 409 忽略 |
| **routing** | 写入与点查带同一 `routing=logId`，降低扫分片 |

伪流程：

```text
miss 后：
  if tryLock("backfill:"+logId):
      body = file.get(logId)
      es.index(_id=logId, body)
      unlock
  else:
      sleep 短暂 / 轮询 es.get
      或自己读文件返回但不写 ES（降级）
```

### 3.3 Nginx 能不能「合并同 logId」？

**标准 Nginx 不能把多个相同 URI 的请求合成一次上游调用。**  
同 logId 的惊群，**必须在应用层**做 singleflight/锁；Nginx 负责的是**别让总流量把入口打垮**（见第 6 节）。

若要用缓存挡重复读：可用 `proxy_cache` 按 URI 缓存**成功响应**（见 6.4），在短 TTL 内减轻 API/ES，但 **miss 回源并发**仍靠后端互斥最稳。

---

## 4. ES 查询慢

点查 logId 应走 **Get by id**，不要用重 search。

| 原因 | 处理 |
|------|------|
| `match`/`query_string` 扫全文 | 改为 `GET /idx/_doc/{logId}` |
| `_source` 塞了整段巨大日志 | ES 只存摘要 + `fileUri`；全文回源文件 |
| 无 routing，扫全部分片 | 读写统一 `routing=logId` |
| 跨很多历史索引查 | 热数据放热索引别名；过期用 ILM 删旧索引 |
| 集群被回填打满 | 回填限速；Bulk；错峰 |
| 列表深分页 | 与点查分离；`search_after` + 强制时间窗 |

---

## 5. ES 过期时间怎么设

ES 日志热层**优先用「按时间索引 + ILM」**，而不是给每个 doc 设 Redis 式 TTL。

### 5.1 推荐：滚动索引 + ILM

```text
logs-2026.10.18
logs-2026.10.19   ← 回填写入当前热索引（别名 logs-hot）
...
超过 N 天 → ILM delete 整索引
```

再点查 miss → 再回源回填，形成闭环。

### 5.2 N 怎么选

| 因素 | 倾向 |
|------|------|
| 命中率高、回源文件贵/慢 | N 大（14～30 天） |
| ES 磁盘紧、文件读便宜 | N 小（3～7 天） |
| 业务「热查只要一周」 | N=7，对齐产品 |

结合指标调：`es_hit_rate`、回源 QPS、磁盘水位。

### 5.3 备选

- 文档带 `indexedAt`，定时 `delete_by_query`（比删索引重）  
- 逻辑过期：读时看 `cachedAt`，过期可当 miss 或异步刷新  

---

## 6. Nginx：多用户查询 & 同 logId 怎么设置

职责边界：

| 诉求 | Nginx | 应用 |
|------|-------|------|
| 很多用户同时查（总流量） | **限流、连接数、超时** | 线程池/连接池 |
| 同一 logId 并发 miss 回填 | 可辅助短缓存 | **互斥 / 幂等（主）** |
| 鉴权 | 可粗拦 | JWT/权限（主） |
| 路由 | `location` 转发 | 业务路由 |

### 6.1 多用户查询：限制总入口流量

防止全体用户把 API/ES/文件服务打满：

```nginx
# 限制每个 IP 的请求速率（多用户场景的基础保护）
limit_req_zone $binary_remote_addr zone=log_by_ip:10m rate=10r/s;

# 限制并发连接（可选）
limit_conn_zone $binary_remote_addr zone=conn_by_ip:10m;

upstream log_api {
    server 127.0.0.1:8080;
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name log.example.com;

    location /api/logs/ {
        limit_req zone=log_by_ip burst=20 nodelay;
        limit_conn conn_by_ip 20;

        proxy_pass http://log_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;

        proxy_connect_timeout 2s;
        proxy_send_timeout 5s;
        proxy_read_timeout 5s;   # 与后端超时预算对齐
    }
}
```

说明：

- `rate=10r/s` + `burst`：按 **IP** 平滑多用户流量；内部办公网可改大或按账号限流（见下）  
- 超时要**短于/对齐**后端对文件服务的预算，避免 Nginx 已断、后端还在回源  

### 6.2 按登录用户限流（多用户更公平）

只按 IP 时，NAT 下多人会误伤。可把用户维度传给 Nginx：

```nginx
# 假设鉴权后由 auth 子请求或网关注入；示意用 header
map $http_x_user_id $limit_key {
    default $http_x_user_id;
    ""      $binary_remote_addr;   # 未登录回落 IP
}

limit_req_zone $limit_key zone=log_by_user:10m rate=5r/s;
```

更常见初创做法：**Nginx 按 IP 粗限流 + 应用按 userId 细限流**。

### 6.3 同一 logId：Nginx 侧能做的

**1）按 URI 限流（防单个 id 被刷）**

```nginx
# 对「具体日志路径」单独限速，避免同一 logId 被打爆
limit_req_zone $request_uri zone=log_by_uri:10m rate=20r/s;

location ~ ^/api/logs/[^/]+$ {
    limit_req zone=log_by_ip burst=20 nodelay;
    limit_req zone=log_by_uri burst=40 nodelay;  # 叠加：同 id 全网共享额度
    proxy_pass http://log_api;
    # ... 同上 proxy 头与超时
}
```

注意：`$request_uri` 含 query string 时要统一；纯 `/api/logs/{id}` 最合适。  
这是**限流**，不是合并回源；后端仍要 singleflight。

**2）短时 proxy_cache（可选，挡重复命中）**

仅缓存**成功的 GET**，减轻重复打 API（尤其已回填后）：

```nginx
proxy_cache_path /var/cache/nginx/logs levels=1:2 keys_zone=log_cache:20m
                 max_size=2g inactive=10m use_temp_path=off;

location ~ ^/api/logs/[^/]+$ {
    proxy_cache log_cache;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    proxy_cache_valid 200 30s;      # 短 TTL；权限敏感慎用
    proxy_cache_methods GET HEAD;
    add_header X-Cache-Status $upstream_cache_status;

    # 有用户态权限时：要么不缓存，要么 cache key 带上 user_id
    # proxy_cache_key "$http_x_user_id:$request_uri";

    limit_req zone=log_by_ip burst=20 nodelay;
    proxy_pass http://log_api;
}
```

权限注意：

- 日志若**按用户隔离**，`proxy_cache_key` **必须带 userId/角色**，否则会串数据  
- 不确定时：**关闭 proxy_cache**，只靠应用限流 + ES  

**3）不要指望 Nginx 解决「并发回填」**  
锁与 singleflight 放在 API；Nginx 只降总 QPS、降同 URI 毛刺。

### 6.4 多用户 + 同 logId 的分工小结

```text
多用户同时查不同日志
  → Nginx：per-IP / per-user 限流 + 超时 + keepalive
  → API：正常点查 ES；miss 各自回源（可全局限流回源）

多用户同时查同一个 logId
  → Nginx：per-URI 限流 +（可选）短 proxy_cache
  → API：singleflight/分布式锁 + ES _id 幂等写  ← 解决惊群回源的关键
```

---

## 7. 端到端时序（同 logId 三用户同时 miss）

```text
UserA/B/C → Nginx（URI 限流通过）
         → API 均 ES miss
         → A 拿到 backfill 锁 → 读文件 → 写 ES → 返回
         → B/C 等锁或轮询 ES → hit → 返回
         （若开了 proxy_cache，后续短时间直接在 Nginx 命中）
```

---

## 8. 难点汇总（面试用）

| 难点 | 对策 |
|------|------|
| 并发 miss 打穿文件服务 | 互斥回填 + `_id` 幂等 |
| 多用户总流量过大 | Nginx `limit_req` / `limit_conn` + 应用限流 |
| 单 logId 被刷 | Nginx 按 `$request_uri` 限流 |
| ES 点查慢 | Get by id、routing、控 `_source`、ILM 控索引体量 |
| ES 过期 | 按天索引 + ILM 保留 N 天 |
| 缓存串权 | 带权限则勿盲目 proxy_cache，或 key 含 userId |
| ES 写失败 | 仍返回文件内容；指标告警 |
| 穿透非法 id | 校验 + 负缓存 |

---

## 9. 面试收口

> 大文件是只写真相；ES 是可 ILM 过期的热层。读路径 Cache-Aside，点查走 `_id`。同 logId 并发 miss 靠应用层互斥与幂等回填；Nginx 用 IP/用户限流扛多用户，用 URI 限流（和可选短缓存）抑制单 id 毛刺，但不替代后端 singleflight。ES 过期用滚动索引 + ILM，按命中率与磁盘选保留天数。

---

## 10. 相关文档

- [02-interview-qa.md](./02-interview-qa.md) — 缓存击穿、限流、网关相关题  
- [08-l4-distributed-resilience-performance.md](./08-l4-distributed-resilience-performance.md) — 弹力与限流  
- [06-l2-professional-fundamentals.md](./06-l2-professional-fundamentals.md) — Cache-Aside  
