# Redis 开发、调优、迁移与备份实战指南

Redis 是高性能内存数据存储，常被用于缓存、会话、限流、排行榜、消息流与实时状态。它并不是关系型数据库的通用替代品：订单、余额、账务等需要复杂查询、强约束和长期可靠保存的数据，通常仍应以 PostgreSQL、MySQL 等持久化数据库为事实来源（source of truth）。

本文覆盖 Redis 7 及以上常用能力，聚焦日常开发、真实产品架构、性能调优、迁移、备份与恢复。所有生产参数都应在与实际负载接近的预生产环境验证后再上线。

## 1. 基础概念与工具

### 1.1 Redis 的数据模型

Redis 将所有数据存放在逻辑数据库的键空间中。键是二进制安全的字符串，值可为多种原生数据结构。

| 类型 | 常见用途 | 常用命令 |
| --- | --- | --- |
| String | 缓存对象、计数器、分布式标记 | `GET`、`SET`、`INCR` |
| Hash | 用户资料、商品摘要等字段集合 | `HSET`、`HGETALL` |
| List | 简单队列、最近访问记录 | `LPUSH`、`BRPOP` |
| Set | 去重、标签、共同关注关系 | `SADD`、`SINTER` |
| Sorted Set | 排行榜、按时间或分数排序 | `ZADD`、`ZRANGE` |
| Stream | 可追踪的消息流和消费者组 | `XADD`、`XREADGROUP` |
| Bitmap | 签到、活跃标记、布尔状态统计 | `SETBIT`、`BITCOUNT` |
| HyperLogLog | 近似 UV 去重计数 | `PFADD`、`PFCOUNT` |
| Geospatial | 附近门店、配送员位置 | `GEOADD`、`GEOSEARCH` |

Redis 命令对单个命令通常是原子执行的，但这不意味着多个命令构成原子事务。需要将“检查再修改”合并为单条命令、使用 `WATCH`/`MULTI`，或使用 Lua 脚本/Redis Functions。

### 1.2 安装、连接与 CLI

本地开发可使用容器快速启动。生产环境应使用受支持的镜像版本、专用持久化卷、密钥管理和受控配置，而不是直接沿用以下示例。

```bash
docker run --name redis-dev -d -p 6379:6379 \
  redis:7 redis-server --appendonly yes

redis-cli -h 127.0.0.1 -p 6379 PING
# PONG
```

常用命令：

```bash
# 使用 URI 连接，密码来自环境变量或安全凭据系统，不写入 Shell 历史
redis-cli -u "rediss://app_user:${REDIS_PASSWORD}@cache.example.com:6380/0" --tls PING

# 获取服务与持久化信息
redis-cli INFO server
redis-cli INFO memory
redis-cli INFO persistence

# 慢查询与延迟诊断
redis-cli SLOWLOG GET 20
redis-cli --latency-history -h cache.example.com -p 6379

# 谨慎使用，仅在非生产或受控条件下检查键空间
redis-cli --scan --pattern 'app:session:*' | head -n 20
```

`KEYS *` 会遍历整个键空间并阻塞主线程，不得在生产库中使用。需要遍历时使用渐进式 `SCAN`，并限制每批数量和处理速率。

### 1.3 关键术语

| 术语 | 含义 |
| --- | --- |
| TTL | 键的剩余存活时间；过期键会被惰性或主动删除 |
| RDB | 某时刻内存数据的紧凑快照文件 |
| AOF | 将写操作追加到文件以便重放的持久化日志 |
| Replication | 主节点将命令流异步复制给副本节点 |
| Sentinel | 监控、自动故障转移和客户端服务发现组件 |
| Redis Cluster | 通过 16,384 个哈希槽分片、可横向扩展的集群模式 |
| Eviction | 在 `maxmemory` 限制下，按策略淘汰键以腾出内存 |

## 2. 安全、命名与数据生命周期

### 2.1 最小安全基线

1. Redis 不应暴露在公网；通过 VPC、安全组和防火墙限制应用网段。
2. 使用 TLS，客户端校验证书；云服务优先使用私网端点。
3. 使用 ACL 创建不同用途的用户，禁止所有应用共用默认用户或管理员密码。
4. 禁用或严格限制 `FLUSHALL`、`FLUSHDB`、`CONFIG`、`MODULE`、`DEBUG` 等高风险命令。
5. 配置 `protected-mode yes`，不要将 `bind 0.0.0.0` 作为通用方案。
6. Redis 文件、备份和 AOF/RDB 存储都应设置最小权限与静态加密。
7. 密码、TLS 私钥和 ACL 文件应从密钥管理系统注入，禁止提交到代码库。

ACL 示例：

```text
# aclfile 中由平台管理员维护；示例密码占位符不可直接使用
user app_cache on >replace-with-secret ~app:* +@read +@write -@dangerous
user app_worker on >replace-with-secret ~app:stream:* ~app:job:* +@read +@write
user readonly on >replace-with-secret ~app:* +@read
```

启用 ACL 后，应用按功能使用独立账号与键前缀。例如，API 服务不需要管理 Stream 消费者组时，就不应获得相关命令权限。

### 2.2 键命名规范

键名应体现业务域、实体、标识和版本，便于 ACL 限制、排障和有序迁移。

```text
app:product:v1:8472
app:user:v1:profile:10086
app:session:v1:9c1f... 
app:ratelimit:v1:login:203.0.113.8:20260930T1200
app:leaderboard:v1:weekly:2026-W40
app:stream:v1:payment-events
```

建议：

- 以冒号分段；避免超长键名和把完整 JSON 放进键名。
- 为可演进的数据结构标注版本，变更时可并行读写 `v1` 与 `v2`。
- 所有临时数据都应明确 TTL；永久键需要有所有者、容量预算和清理机制。
- 同一对象的大型集合要设置边界，例如只保留最近 1,000 条记录。
- 在 Redis Cluster 中，必须一起执行多键原子操作时使用哈希标签，如 `cart:{user:10086}:items` 与 `cart:{user:10086}:meta`。花括号中的部分决定槽位。

### 2.3 TTL 与内存预算

```bash
redis-cli SET app:product:v1:8472 '{"name":"..."}' EX 300 NX
redis-cli TTL app:product:v1:8472
redis-cli EXPIRE app:product:v1:8472 300
redis-cli PERSIST app:product:v1:8472
```

生产系统应在设计阶段给每类键建立预算：

$$
Memory\ Budget \approx Key\ Count \times (Average\ Key\ Bytes + Average\ Value\ Bytes + Redis\ Overhead)
$$

实际占用还包括哈希表、对象元数据、碎片、复制缓冲区和持久化时的写时复制（COW）额外内存。应通过压测及 `MEMORY USAGE`、`INFO memory` 实测，而不是只根据原始 JSON 大小估算。

```bash
redis-cli MEMORY USAGE app:product:v1:8472
redis-cli --bigkeys
```

`--bigkeys` 仅适用于受控排查，它会扫描键空间并消耗资源。对大实例，应在副本或低峰期执行，或改用采样监控。

## 3. 日常开发使用

### 3.1 缓存模式：Cache-Aside

真实产品最常用的是 Cache-Aside：先读缓存，未命中时读取主数据库，再写入缓存。写路径先更新主数据库，提交成功后删除缓存，使后续读请求回源重建。

```text
读路径：应用 -> GET Redis -> 命中则返回
                       -> 未命中则查询主库 -> SET Redis（带 TTL）-> 返回

写路径：应用 -> 更新主库并提交 -> DEL Redis 对应键 -> 返回
```

伪代码：

```typescript
const key = `app:product:v1:${productId}`;
const cached = await redis.get(key);
if (cached) return JSON.parse(cached);

const product = await database.products.findById(productId);
if (!product) return null;

const ttlSeconds = 300 + randomInteger(0, 60);
await redis.set(key, JSON.stringify(product), { EX: ttlSeconds });
return product;
```

```typescript
await database.transaction(async (transaction) => {
  await transaction.products.update(productId, changes);
});
await redis.del(`app:product:v1:${productId}`);
```

写后删除而非直接覆盖缓存，能减少多写入方、序列化版本不一致和字段遗漏带来的风险。删除与后续重建之间存在短暂不一致窗口；对于强一致业务，应直接读取主库或采用版本号、消息驱动失效和业务补偿，而不能把 Redis 当作唯一事实来源。

### 3.2 缓存常见风险与治理

| 问题 | 表现 | 处理方式 |
| --- | --- | --- |
| 缓存穿透 | 大量查询不存在的数据，持续打到主库 | 缓存空值并设置较短 TTL；做参数校验；必要时使用布隆过滤器 |
| 缓存击穿 | 热点键过期瞬间并发回源 | 互斥重建、逻辑过期、请求合并、预热 |
| 缓存雪崩 | 大量键同时过期或缓存整体不可用 | TTL 加随机抖动、多级缓存、限流降级、隔离与容量冗余 |
| 缓存污染 | 低复用大对象挤占热点数据 | 设置最大对象/集合边界，审查 TTL 和淘汰策略 |
| 一致性漂移 | 缓存与主库数据不一致 | 写后失效、事件驱动失效、版本校验、定期校验 |

逻辑过期适用于允许返回短暂旧数据的场景：值中携带 `refreshAfter`，过期后只有一个请求异步重建，其他请求继续读取旧值。它不适用于库存、风控、权限撤销等必须立即生效的数据。

### 3.3 Hash、Set、Sorted Set 的业务用法

```bash
# Hash：用户资料的少量字段
redis-cli HSET app:user:v1:profile:10086 nickname alice level 12 updated_at 1720000000
redis-cli HGETALL app:user:v1:profile:10086

# Set：幂等去重或标签集合
redis-cli SADD app:campaign:v1:eligible:20260930 10086
redis-cli SISMEMBER app:campaign:v1:eligible:20260930 10086

# Sorted Set：周排行榜，分数为累计积分
redis-cli ZINCRBY app:leaderboard:v1:weekly:2026-W40 10 10086
redis-cli ZREVRANGE app:leaderboard:v1:weekly:2026-W40 0 99 WITHSCORES
```

实际产品中的排行榜应在活动结束后归档或设置 TTL，避免无限增长。若需要同时更新积分、记录明细和发奖，Redis 排名仅用于展示或快速筛选，最终结算应由持久化数据库或事件日志完成。

### 3.4 原子计数、限流与去重

固定窗口限流的简单实现：

```bash
redis-cli INCR app:ratelimit:v1:login:203.0.113.8:20260930T1200
redis-cli EXPIRE app:ratelimit:v1:login:203.0.113.8:20260930T1200 120 NX
```

`INCR` 和首次设置 TTL 若分两次调用，在故障时可能留下永不过期的键。可使用 Lua 脚本将其合并为原子操作：

```lua
-- KEYS[1] = rate-limit key, ARGV[1] = window seconds
local current = redis.call('INCR', KEYS[1])
if current == 1 then
  redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return current
```

生产级滑动窗口或令牌桶限流还应返回剩余额度和重试时间。Redis Cluster 中，脚本涉及的多个键必须位于相同哈希槽，通常通过同一个哈希标签实现。

幂等键示例：

```bash
# 对同一支付请求只允许一个执行者占位；值保存处理状态或请求摘要
redis-cli SET app:idempotency:v1:payment:request-uuid processing NX EX 86400
```

占位成功才执行支付逻辑；完成后存储响应摘要。关键支付状态仍必须持久化到事务数据库，Redis 宕机或键过期不能导致重复扣款。

### 3.5 事务、Lua 与 Redis Functions

`MULTI`/`EXEC` 会按顺序执行队列中的命令，但不支持关系数据库式自动回滚；运行时命令错误不会撤销已成功命令。

```bash
redis-cli
MULTI
INCR app:counter:v1:page-view
EXPIRE app:counter:v1:page-view 86400
EXEC
```

对依赖当前值的条件更新，Lua 脚本可在单线程中原子执行：

```lua
-- 仅当库存足够时扣减。KEYS[1] = stock key, ARGV[1] = amount
local stock = tonumber(redis.call('GET', KEYS[1]) or '0')
local amount = tonumber(ARGV[1])
if stock < amount then
  return {err = 'INSUFFICIENT_STOCK'}
end
return redis.call('DECRBY', KEYS[1], amount)
```

脚本必须短小、确定且没有网络 I/O、长循环或全量扫描。长脚本会阻塞所有请求；应设置 `lua-time-limit`、通过慢日志监控，并将复杂业务编排移到应用服务。Redis Functions 适合部署可复用逻辑，但也要版本化、灰度和审计。

### 3.6 分布式锁：适用边界

简单锁获取：

```bash
# token 必须是不可预测的唯一值，而不是固定字符串
redis-cli SET app:lock:v1:report:20260930 unique-random-token NX PX 30000
```

释放锁必须校验 token，避免客户端 A 的过期锁被客户端 B 获取后，A 再误删 B 的锁：

```lua
-- KEYS[1] = lock key, ARGV[1] = token
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
end
return 0
```

Redis 锁适合降低重复工作、控制缓存重建或协调非关键任务。对于库存扣减、资金转移、全局唯一写入等必须证明正确性的场景，优先使用数据库约束、事务、队列分区或带 fencing token 的协调机制。不要仅因 Redis 锁存在，就假设网络分区、暂停和主从切换下仍能保证强互斥。

## 4. Redis 在真实产品中的使用方式

### 4.1 电商和内容产品

| 场景 | Redis 的职责 | 事实来源与关键约束 |
| --- | --- | --- |
| 商品详情 | 缓存商品、价格、库存展示摘要 | 商品/价格在主数据库；价格变更后删除或事件失效缓存 |
| 购物车 | 保存未登录或短期活跃购物车 | 下单前持久化校验价格、库存和优惠；设置闲置 TTL |
| 秒杀资格 | 预过滤、排队令牌、限流 | 订单创建和扣减由可靠事务/队列处理，避免超卖 |
| 首页 Feed | 缓存聚合结果、候选集合 | 推荐系统和内容库为事实来源；分用户/版本设置 TTL |
| 排行榜 | Sorted Set 实时排序 | 活动结束落库、冻结结果并审计发奖依据 |

一个典型商品详情读取链路：CDN 缓存静态图片，应用进程内短缓存承接极热数据，Redis 承担共享对象缓存，未命中再访问数据库。商品发布事件触发 Redis 缓存删除或版本更新。这样即使 Redis 发生故障，也能通过限流和降级回源，而不是让业务整体不可用。

### 4.2 身份、会话与权限

Redis 很适合保存服务器端 session、短期验证码、单点登录会话和权限版本号：

```text
app:session:v1:<opaque-session-id> -> {userId, deviceId, issuedAt, authVersion}, TTL 30m
app:user:v1:auth-version:<userId> -> integer
app:otp:v1:login:<normalized-phone> -> hash, TTL 5m
```

用户改密、注销全部设备或权限撤销时，递增 `auth-version` 或删除对应会话键。鉴权请求比较令牌中携带的版本与当前版本，能让旧令牌快速失效。高安全系统仍应考虑 Redis 不可用时的策略：例如短暂拒绝高风险操作、回退到身份库，或将必要会话持久化。

### 4.3 API 网关与反滥用

网关常用 Redis 按用户、API Key、IP、设备或租户维度做限流、配额和并发控制。实现时要防止单个热点键成为集群瓶颈，可通过按租户或时间分片，并将异常请求在边缘层尽早拦截。

限流键中的 IP、用户 ID 等外部输入必须先规范化和限制长度，避免攻击者构造无限键空间消耗内存。对登录、验证码、密码重置等端点还应结合设备指纹、行为规则和审计，Redis 计数器只是其中一个信号。

### 4.4 实时状态、消息与任务

| 能力 | 合适使用 Redis 的情形 | 不应忽略的限制 |
| --- | --- | --- |
| 在线状态 | 保存用户/设备心跳、房间成员，设置短 TTL | 心跳丢失与网络抖动会造成短暂不准 |
| Pub/Sub | WebSocket 节点间实时通知、可丢失提醒 | 不持久化，断线订阅者会丢消息 |
| Streams | 消费者组、重试、待确认消息、轻量事件处理 | 要监控 pending、裁剪流长度和消费者故障 |
| List 队列 | 简单后台任务 | 复杂重试、顺序、审计和长期保留需求更适合专用消息系统 |

Redis Pub/Sub 适合“新消息提示”“缓存失效通知”等可丢失事件，不适合订单、扣款、发货等必须可靠交付的事实事件。对可靠工作流，可使用 Redis Streams 配合消费者组，或采用 Kafka、RabbitMQ、云消息队列，并将业务状态持久化。

Stream 消费者组基本流程：

```bash
# 创建流和组；MKSTREAM 允许在不存在时创建
redis-cli XGROUP CREATE app:stream:v1:email-jobs workers '$' MKSTREAM

# 生产者写入任务，MAXLEN 约束长度避免无限增长
redis-cli XADD app:stream:v1:email-jobs MAXLEN '~' 100000 '*' \
  type welcome user_id 10086 template v2

# 消费者读取新消息；处理成功后必须确认
redis-cli XREADGROUP GROUP workers worker-01 COUNT 10 BLOCK 5000 \
  STREAMS app:stream:v1:email-jobs '>'
redis-cli XACK app:stream:v1:email-jobs <message-id>
```

消费者应具备幂等性。对长期未确认消息，需要用 `XPENDING`、`XAUTOCLAIM` 等机制转交给健康消费者，并将失败任务转移至可审查的死信流程。

### 4.5 实时分析和地理位置

Bitmap 可存储“某用户某天是否活跃”，HyperLogLog 可近似统计 UV，Geo 可检索附近服务点。这些能力适合实时看板、粗粒度运营决策和附近资源发现；但涉及结算、合规报表或精确审计时，应将原始事件进入可靠数仓/日志系统。

```bash
# 记录某个用户当天的活跃位（用户编号需受控映射）
redis-cli SETBIT app:active:v1:2026-09-30 10086 1
redis-cli BITCOUNT app:active:v1:2026-09-30

# 附近门店，坐标为经度、纬度
redis-cli GEOADD app:store:v1:locations 116.397128 39.916527 beijing-001
redis-cli GEOSEARCH app:store:v1:locations FROMLONLAT 116.40 39.91 BYRADIUS 5 km ASC COUNT 20
```

## 5. 高可用、分片与客户端设计

### 5.1 单实例、Sentinel 与 Cluster 的选择

| 部署方式 | 适用场景 | 优点 | 限制 |
| --- | --- | --- | --- |
| 单实例 | 本地开发、小型非关键缓存 | 简单、低成本 | 单点故障、容量有限 |
| 主从 + Sentinel | 高可用但总内存可容纳于单主节点 | 自动故障转移、运维相对成熟 | 不横向分片；故障转移可能丢最后一小段写入 |
| Redis Cluster | 大内存、吞吐横向扩展、需要分片 | 自动分片和副本故障转移 | 多键操作受哈希槽限制，运维与客户端要求更高 |
| 托管 Redis | 大多数生产业务 | 托管监控、备份、故障处理 | 仍需理解参数、容量、网络和恢复语义 |

生产建议至少一个主节点和一个副本，并跨可用区部署。Redis 复制默认异步，因此主节点故障转移时可能丢失尚未复制的最近写入。`WAIT` 可以等待一定数量副本确认，从而降低窗口，但它不是强一致共识协议。

```bash
# 写入后最多等待 1 秒，要求至少一个副本确认
redis-cli SET app:critical-cache:v1:42 value EX 300
redis-cli WAIT 1 1000
```

对不能接受丢失的业务，不能仅依赖 Redis 主从复制；应在持久化事务系统中建立可靠记录。

### 5.2 客户端连接原则

- 使用官方或成熟客户端，启用连接池、超时、TLS、ACL 和 Sentinel/Cluster 拓扑发现。
- 建立连接、命令、读取和总请求超时；重试只适用于已证明幂等的操作。
- 对 Redis 故障设置快速失败、熔断、限流和降级，避免请求无限堆积。
- 使用 pipeline 合并独立命令，减少网络往返；不要将有条件的相关写入误认为 pipeline 原子化。
- Cluster 客户端必须处理 `MOVED` 和 `ASK` 重定向，通常由成熟客户端自动完成。
- 避免在请求路径中序列化/反序列化超大对象；设置值大小、集合长度和单次返回数量上限。

## 6. 监控、诊断与性能调优

### 6.1 必须监控的指标

| 维度 | 关键指标 | 风险信号 |
| --- | --- | --- |
| 可用性 | 成功率、连接失败、角色变更 | Sentinel/Cluster 不稳定、频繁切换 |
| 延迟 | P50/P95/P99、命令耗时、事件循环延迟 | 尾延迟突升、慢命令增加 |
| 内存 | `used_memory`、RSS、碎片率、淘汰数 | 接近上限、持续淘汰、碎片异常 |
| 负载 | ops/s、网络流量、CPU、命中率 | 热点键、单线程饱和、网络拥塞 |
| 持久化 | RDB/AOF 最近成功时间、fork 耗时、rewrite 状态 | 备份失败、AOF 重写过慢、COW 内存不足 |
| 复制 | 副本延迟、复制偏移、全量同步次数 | 复制链路断开、频繁 full resync |
| 键空间 | 过期数、keyspace hits/misses、大键 | TTL 异常、穿透、内存增长失控 |

常用现场检查：

```bash
redis-cli INFO all
redis-cli INFO stats
redis-cli INFO replication
redis-cli INFO keyspace
redis-cli CONFIG GET maxmemory maxmemory-policy appendonly save
redis-cli SLOWLOG LEN
redis-cli LATENCY DOCTOR
```

`INFO all` 输出较大，建议通过监控系统周期采集；人工排障时先查看具体 section，避免无目的地抓取和转储敏感信息。

### 6.2 慢查询与阻塞问题

Redis 主线程按顺序处理大多数命令。单条耗时命令、巨大值序列化、全量扫描、耗时 Lua 脚本、AOF 重写和系统内存交换都可能拉高尾延迟。

```bash
# 查看慢日志阈值与最近记录
redis-cli CONFIG GET slowlog-log-slower-than slowlog-max-len
redis-cli SLOWLOG GET 50

# 在受控环境运行内置延迟诊断
redis-cli --latency
redis-cli --intrinsic-latency 100
```

排查优先级：

1. 识别慢命令名称、参数大小、调用方和键模式。
2. 检查是否使用 `KEYS`、大范围 `SMEMBERS`/`HGETALL`、无界 `LRANGE`/`ZRANGE` 或长 Lua 脚本。
3. 确认是否存在大键、热键、网络瓶颈、CPU 抢占、内存交换或磁盘延迟。
4. 检查 RDB/AOF fork 期间是否出现 COW 内存压力和延迟尖峰。
5. 改为分页/游标访问、限制集合、分片热点或异步化批量工作，并重新压测验证。

### 6.3 内存、淘汰与碎片

Redis 不应依赖操作系统 OOM 来管理容量。必须设置 `maxmemory` 并根据业务选择淘汰策略。

| 策略 | 含义 | 适用建议 |
| --- | --- | --- |
| `noeviction` | 达到上限时写入报错 | 不能默默丢缓存，但应用必须处理写失败 |
| `allkeys-lru` | 从所有键中近似淘汰最久未使用键 | 通用缓存常见选择 |
| `allkeys-lfu` | 从所有键中近似淘汰最不常用键 | 访问频率差异大的缓存 |
| `volatile-ttl` | 仅从设置 TTL 的键中优先淘汰即将过期键 | 所有可淘汰缓存键都必须有 TTL |
| `volatile-lru`/`volatile-lfu` | 仅在带 TTL 键中按近似 LRU/LFU 淘汰 | 永久键与缓存键混用时需特别小心 |

```conf
# redis.conf 示例，必须与业务写失败策略一起验证
maxmemory 12gb
maxmemory-policy allkeys-lfu
maxmemory-samples 10
```

碎片率可由 $used\_memory\_rss / used\_memory$ 粗略衡量。它偏高不一定代表故障，需结合 allocator、负载模式和 RSS 变化判断。频繁的大对象分配释放会增加碎片；可通过限制对象大小、优化数据结构、使用主动碎片整理（在支持且验证后）或滚动重启节点处理。绝不能在内存不足时直接重启唯一主节点来“释放内存”。

### 6.4 数据结构与命令优化

- 用 `HASH` 存小型同类字段通常比每个字段独立键更节省内存；超大 Hash 仍需拆分。
- 小型集合会使用紧凑编码，超过配置阈值会转换为更通用结构；不要依赖内部编码作为业务契约。
- 读取大集合使用 `HSCAN`、`SSCAN`、`ZSCAN`；范围查询使用分页和明确 `LIMIT`。
- 使用 `UNLINK` 异步释放非常大的键，避免 `DEL` 在主线程阻塞；仍须评估后台释放的内存压力。
- pipeline 用于独立批量请求；每批大小通过压测确定，过大 pipeline 会消耗客户端与服务端内存。
- 不要用 `MONITOR` 作为长期生产观察工具，它会产生大量开销并可能泄露数据。

### 6.5 持久化参数与 COW 风险

RDB 快照和 AOF 重写会 fork 子进程。fork 后若主进程持续修改大量内存页，操作系统将复制这些页面形成 COW，短时间内内存可能显著上升。高写入实例应为 COW 预留内存，并观察 fork 耗时与持久化延迟。

```conf
# 典型混合持久化示例，值需根据 RPO/RTO 和写入量调整
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# RDB 作为额外快照；不要将此示例视作所有环境的固定标准
save 3600 1
save 300 100
save 60 10000
```

`appendfsync always` 的数据耐久性更强但延迟和磁盘压力更高；`everysec` 通常是在性能和最坏约约 1 秒数据风险之间的折中。主从异步复制、AOF 刷盘、云块存储和应用确认语义共同决定实际 RPO，不能只看一个参数。

## 7. 备份、恢复与灾难演练

### 7.1 RDB、AOF 与副本不是一回事

| 机制 | 优点 | 风险或限制 |
| --- | --- | --- |
| RDB 快照 | 紧凑、恢复较快、适合定期备份 | 两次快照间的数据可能丢失 |
| AOF | 记录写操作，RPO 可更低 | 文件更大、恢复可能更慢，需要重写管理 |
| RDB + AOF | 兼顾恢复速度和数据完整性 | 配置、容量和监控更复杂 |
| 副本 | 读扩展与故障转移 | 异步复制会传播误删，不能替代备份 |
| 托管服务快照 | 运维便利、可跨区域 | 需验证保留策略、恢复权限和实际 RTO |

Redis 7 的 AOF 可采用多段文件和 manifest 管理。不要手工只复制某一个增量 AOF 文件；应以备份工具、托管服务快照或一致性文件集为单位保存。

### 7.2 备份策略

1. 按 RPO 定义快照频率、AOF 策略和异地复制策略。
2. 定期在低峰或副本节点生成 RDB，避免对主节点增加不必要的 fork 压力。
3. 将 RDB/AOF 或托管快照复制到独立、加密、不可随业务账号删除的对象存储。
4. 设置保留周期、校验、备份失败和“最后成功备份年龄”告警。
5. 记录实例版本、配置、ACL、TLS 证书链、Cluster 拓扑和数据恢复依赖。
6. 至少季度演练恢复，测量真实 RTO 并验证关键数据和应用行为。

手动触发与检查：

```bash
# 后台生成 RDB 快照；不要频繁在高写入主节点随意执行
redis-cli BGSAVE
redis-cli LASTSAVE
redis-cli INFO persistence

# 检查 AOF 文件完整性（在备份副本上操作文件）
redis-check-aof --fix /backup/appendonly.aof
```

`redis-check-aof --fix` 会截断损坏尾部，应只对副本执行，并先保留原始文件。任何“修复”都可能丢弃最后一段无法解析的写操作。

### 7.3 恢复步骤

恢复必须在隔离环境先验证，绝不直接覆盖唯一生产实例：

1. 停止目标 Redis 服务，保留其现有数据目录快照和配置。
2. 确认备份版本兼容、校验通过，并准备足够磁盘和内存。
3. 将完整 RDB 文件放入目标 `dir`，名称匹配 `dbfilename`；或恢复完整 AOF 文件集及 manifest。
4. 使用隔离端口、隔离网络、只读/测试 ACL 启动实例。
5. 检查启动日志、`INFO persistence`、键数量、内存、关键 TTL、消费者组 pending 和抽样业务数据。
6. 应用侧冒烟测试通过后，按变更计划切换连接；保留旧实例以便回退和审计。

RDB 恢复示例：

```conf
# redis-restore.conf，仅用于隔离恢复实例
port 16379
bind 127.0.0.1
protected-mode yes
dir /restore/redis
dbfilename dump.rdb
appendonly no
```

```bash
cp /secure-backup/redis-prod-2026-09-30.rdb /restore/redis/dump.rdb
redis-server /etc/redis/redis-restore.conf
redis-cli -p 16379 DBSIZE
redis-cli -p 16379 INFO persistence
```

### 7.4 误删与逻辑恢复

Redis 原生命令通常没有表级回收站或时间点恢复。误执行 `DEL`、`FLUSHDB`、错误脚本或错误过期策略后，能否恢复取决于最近可用备份、AOF、复制延迟和操作发现速度。

应急原则：

1. 立即停止有害客户端或撤销其 ACL，记录精确时间和影响键模式。
2. 保护当前节点、AOF、RDB、日志与副本，避免自动重启或继续覆盖证据。
3. 在隔离环境恢复到误操作前的最近备份或可用 AOF 状态。
4. 使用受控的 `SCAN` 和导入脚本抽取需补回的数据，验证后写回生产。
5. 复盘并收紧高危命令、生产权限、审批、审计和恢复演练。

对重要键应设计业务级补偿数据源，例如订单库、事件日志或对象存储，而不应期望 Redis 单独承担可审计的永久记录职责。

## 8. 数据迁移与版本升级

### 8.1 迁移方式选择

| 场景 | 推荐方式 | 特点 |
| --- | --- | --- |
| 小数据量、可短暂停机 | RDB/AOF 备份恢复 | 实现简单，切换期间停止写入 |
| 单实例更换主机 | 复制新副本后提升，或托管迁移服务 | 可缩短停机，需核对复制追平 |
| 大数据量、异构集群 | 专用在线迁移工具或云迁移服务 | 支持全量 + 增量同步，需充分演练 |
| Cluster 扩缩容 | `redis-cli --cluster reshard` / `rebalance` | 在线迁槽，需控制速率并监控热点 |
| 大版本升级 | 新集群复制/迁移后切换 | 不直接拿新二进制启动旧数据目录 |

迁移前必须盘点：Redis 版本和模块、数据量与最大键、持久化配置、ACL、TLS、客户端版本、淘汰策略、Cluster 槽分布、复制延迟、网络带宽、写入峰值与回滚方案。

### 8.2 短暂停机迁移

适用于缓存型数据或允许短暂写入暂停的系统：

1. 预置目标实例、网络、TLS、ACL、容量和监控，确保目标 `maxmemory-policy` 符合业务。
2. 在源端停止业务写入，并等待正在处理的任务完成或进入可恢复状态。
3. 执行最终 RDB/AOF 备份，将完整文件集安全传输到目标。
4. 在隔离端先恢复验证：键数量、关键前缀、TTL、Stream 消费组、应用连接。
5. 切换 DNS、服务发现或配置中心中的 Redis 端点，逐步放量。
6. 保留源实例只读或离线快照，在观察期结束后才清理。

对纯缓存，可选择不搬迁数据并在新集群预热；这往往更简单可靠。前提是主数据库能承受缓存冷启动的回源压力，并已配置限流与降级。

### 8.3 在线迁移与双写验证

大规模或低停机迁移常见路径为“全量复制 -> 增量追平 -> 短暂冻结 -> 切流”。实现可以使用受维护的迁移工具、云厂商迁移服务或在适用场景下建立复制链路。无论工具如何，切换原则一致：

1. 先完成兼容性验证，特别是 Cluster、TLS、ACL、模块命令与数据编码。
2. 迁移期间持续监控延迟、错误率、源端内存、网络带宽和目标端写入能力。
3. 对关键键进行抽样校验：值摘要、TTL、集合基数、Sorted Set 分数和 Stream pending。
4. 切换窗口停止或排队关键写入，等待增量延迟归零。
5. 先灰度少量读流量，再切换写入；只要无法保证双写顺序和失败补偿，就避免长期双写。
6. 建立明确回滚条件。目标已接收独立写入后，简单切回源端会产生数据分叉，必须有补偿策略。

### 8.4 Cluster 扩缩容

Redis Cluster 的数据按哈希槽分布。扩容不是只增加节点，还要将槽迁移到新节点；缩容前必须将待下线节点的槽全部迁出。

```bash
# 仅在演练环境或按明确变更单执行；先检查节点与槽健康
redis-cli --cluster check cluster-node-1.example.com:6379

# 自动平衡槽（实际生产应先指定范围、速率和观察窗口）
redis-cli --cluster rebalance cluster-node-1.example.com:6379
```

迁槽会消耗源、目标节点 CPU、网络与磁盘带宽，也可能加重热点。高峰期避免大规模 re-shard；完成后确认所有槽已覆盖、无迁移中状态、各节点内存均衡且客户端无重定向异常。

### 8.5 版本升级

版本升级前：

- 阅读目标版本 release notes、弃用项和安全公告。
- 核实客户端、模块、RDB/AOF 格式与托管平台的兼容承诺。
- 用生产规模或代表性数据进行恢复、压测、故障切换和回滚演练。
- 保留经验证的备份；定义源集群保留时间和切换后数据回流方案。
- 升级后复查持久化状态、延迟、内存、复制、集群槽和业务错误率。

不要直接对关键生产节点做不可逆原地升级。通常更稳妥的方式是新建目标集群，进行同步或恢复验证后，再通过受控流量切换完成升级。

## 9. 生产变更与日常运行手册

### 9.1 变更前

- 明确缓存、会话、队列、限流等不同键空间的业务影响及数据等级。
- 确认主从健康、复制延迟、备份最新成功时间、磁盘容量和内存余量。
- 在接近生产的环境压测参数、脚本、迁移和故障转移。
- 设置执行窗口、观测指标、停止条件、回滚负责人和应用验证清单。
- 任何涉及 `FLUSH*`、批量删除、`CONFIG SET`、淘汰策略、持久化和 ACL 的操作均应走高风险变更流程。

### 9.2 日常清单

**每日**

- 检查实例可用性、错误率、P95/P99 延迟、客户端连接、CPU、网络与内存余量。
- 检查 `evicted_keys`、命中率、慢日志、新增大键、过期异常和持久化失败。
- 检查副本延迟、复制断开、Cluster 槽覆盖和 Sentinel 故障转移事件。
- 确认备份、AOF/RDB 归档和异地副本在预期时间内成功。

**每周**

- 审核 Top 键空间、容量增长、TTL 分布、热点/大键、淘汰策略和慢命令调用方。
- 验证 ACL、过期账号、生产高危命令限制及 TLS 证书有效期。
- 在隔离环境抽样恢复最近备份，检查键数量、TTL 和关键业务数据。
- 复查应用超时、重试、熔断与降级是否能覆盖 Redis 短暂故障。

**每季度或重大变更后**

- 演练单节点故障、主从切换、Cluster 节点不可用、完整恢复和误删补偿。
- 审核 RPO/RTO、容量预测、跨可用区/跨区域策略及成本。
- 更新版本、模块、客户端依赖与安全补丁计划。
- 复盘慢查询、缓存事故、迁移与故障，修订运行手册和自动化告警。

### 9.3 最小上线验收标准

- Redis 的职责、事实来源、降级路径、键空间所有者和 TTL 策略均已明确。
- 使用 TLS、ACL、私网访问和受管密钥；应用不是以管理员身份连接。
- 已设置 `maxmemory`、经过选择的淘汰策略、容量告警和写失败处理逻辑。
- 已接入延迟、内存、淘汰、命中、慢日志、复制、持久化与备份监控。
- 已验证 RDB/AOF 或托管快照恢复，实测结果满足已确认的 RPO/RTO。
- 客户端有合理超时、幂等重试、连接池、熔断和缓存不可用时的降级策略。
- 对库存、账务、支付、权限撤销等关键场景，Redis 不是唯一数据来源或唯一正确性保障。

## 10. 常用命令速查

```bash
# 基础状态
redis-cli PING
redis-cli DBSIZE
redis-cli INFO memory
redis-cli INFO replication

# 键与 TTL（生产请用 SCAN 代替 KEYS）
redis-cli GET app:product:v1:8472
redis-cli TTL app:product:v1:8472
redis-cli SCAN 0 MATCH 'app:product:v1:*' COUNT 100

# 内存与慢命令
redis-cli MEMORY STATS
redis-cli MEMORY USAGE app:product:v1:8472
redis-cli SLOWLOG GET 20
redis-cli LATENCY LATEST

# 持久化与复制
redis-cli BGSAVE
redis-cli LASTSAVE
redis-cli INFO persistence
redis-cli ROLE

# Cluster 健康检查
redis-cli -c -h cluster-node-1.example.com CLUSTER INFO
redis-cli -c -h cluster-node-1.example.com CLUSTER NODES
```

Redis 的价值在于把高频、低延迟、短生命周期或实时协调类工作从主数据库中分流出来。可靠的产品实践不是“把数据放进 Redis”，而是明确数据权威来源、失效和降级策略、内存边界，以及经过验证的故障恢复能力。