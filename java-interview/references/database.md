# 数据库与中间件 - 面试知识点参考

## 目录
1. [MySQL 索引](#mysql-索引)
2. [MySQL 事务与锁](#mysql-事务与锁)
3. [MySQL 优化](#mysql-优化)
4. [Redis](#redis)
5. [消息队列](#消息队列)

---

## MySQL 索引

### B+ 树索引（高频⭐⭐⭐⭐⭐）

**为什么选 B+ 树而不是 B 树？**
- B+ 树非叶节点不存数据，只存索引键，同等大小下层数更少（通常 3~4 层）
- 叶节点形成有序链表，范围查询效率高
- 数据都在叶节点，查询路径长度稳定

**为什么不用红黑树/跳表？**
- 磁盘 IO 是瓶颈，需要每次读取尽量多的数据（一个页 16KB）
- B+ 树高扇出设计完美匹配磁盘分页读取

### 聚簇索引 vs 非聚簇索引

| 对比 | 聚簇索引（主键）| 非聚簇索引（二级）|
|------|--------------|----------------|
| 叶节点存储 | 完整行数据 | 主键值 |
| 数量 | 一个表只有一个 | 可多个 |
| 回表 | 不需要 | 需要（除覆盖索引）|

**回表**：通过二级索引找到主键值后，再查主键索引获取完整数据的过程

**覆盖索引**：查询的字段全部在索引中，无需回表（高性能查询的关键优化手段）

### 索引失效场景（高频）

```sql
-- 1. 最左前缀原则（联合索引必须从左开始匹配）
INDEX(a, b, c)
WHERE b = 1           -- ❌ a 断了
WHERE a = 1 AND c = 1 -- ⚠️  只用到 a，c 失效

-- 2. 对索引列做函数运算
WHERE YEAR(create_time) = 2024  -- ❌
WHERE create_time >= '2024-01-01' -- ✅

-- 3. 隐式类型转换
WHERE phone = 13812345678  -- ❌ phone 是 varchar，会转换
WHERE phone = '13812345678' -- ✅

-- 4. 使用 != / NOT IN / NOT EXISTS
-- 5. LIKE 以 % 开头：WHERE name LIKE '%张'
-- 6. OR 的左右有一侧无索引
-- 7. 索引列参与运算：WHERE age + 1 = 18
```

---

## MySQL 事务与锁

### 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|----------|------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ |
| READ COMMITTED | ❌ | ✅ | ✅ |
| **REPEATABLE READ**（MySQL 默认）| ❌ | ❌ | ⚠️（Next-Key Lock 部分解决）|
| SERIALIZABLE | ❌ | ❌ | ❌ |

### MVCC（多版本并发控制）（高频⭐⭐⭐⭐）

**解决问题**：读写不互斥，提高并发性能

**核心组件**：
- **隐藏列**：每行数据有 `trx_id`（最近修改事务ID）和 `roll_pointer`（指向 undo log）
- **undo log**：保存历史版本数据，构成版本链
- **Read View**：事务开始时的一致性视图（记录当前活跃事务列表）

**可见性判断**：
```
trx_id < min_trx_id → 版本在 ReadView 创建前提交 → 可见
trx_id > max_trx_id → 版本在 ReadView 创建后 → 不可见
min ≤ trx_id ≤ max → 检查是否在活跃列表 → 在则不可见，不在则可见
```

**RR vs RC 的区别**：
- RC：每次 SELECT 生成新的 Read View（所以能读到已提交的新数据）
- RR：事务开始时生成 Read View，整个事务期间复用（所以可重复读）

### 锁类型

**按粒度**：行锁（Record Lock）、间隙锁（Gap Lock）、临键锁（Next-Key Lock = Record + Gap）

**Next-Key Lock 防止幻读**：
```sql
-- id 索引存在：3, 7, 10
SELECT * FROM t WHERE id = 7 FOR UPDATE;
-- 锁定范围：(3, 7] → 防止其他事务在此范围插入数据
```

**死锁**：两个事务互相等待对方释放锁
- 检测：InnoDB 自动检测，选择回滚代价小的事务
- 预防：按固定顺序加锁、缩短事务、减少锁粒度

---

## MySQL 优化

### EXPLAIN 关键字段

```sql
EXPLAIN SELECT * FROM user WHERE age > 18;
```

| 字段 | 关注点 |
|------|--------|
| `type` | 最重要：system > const > eq_ref > ref > range > index > ALL |
| `key` | 实际使用的索引 |
| `rows` | 预估扫描行数（越小越好）|
| `Extra` | Using filesort / Using temporary 是危险信号 |

**type 优化目标**：至少达到 `range`，最好 `ref`

### 慢查询优化步骤

1. `EXPLAIN` 分析执行计划
2. 检查是否命中索引（`key` 字段）
3. 检查扫描行数（`rows`）
4. 针对性优化：加索引、覆盖索引、改写 SQL、分批查询

### 分页优化

```sql
-- 深分页慢（需扫描 100万+ 行）
SELECT * FROM t ORDER BY id LIMIT 1000000, 10;

-- 优化1：记住上次 ID（适合连续翻页）
SELECT * FROM t WHERE id > last_id ORDER BY id LIMIT 10;

-- 优化2：子查询定位（利用覆盖索引）
SELECT * FROM t WHERE id >= (
    SELECT id FROM t ORDER BY id LIMIT 1000000, 1
) LIMIT 10;
```

---

## Redis

### 五种基本数据类型

| 类型 | 底层实现 | 典型场景 |
|------|---------|---------|
| String | SDS（简单动态字符串）| 缓存、计数器、分布式锁 |
| Hash | ziplist / hashtable | 对象存储（用户信息）|
| List | quicklist（ziplist链表）| 消息队列、最新列表 |
| Set | intset / hashtable | 去重、交并差集 |
| ZSet | ziplist / skiplist+dict | 排行榜、延迟队列 |

### 持久化（高频⭐⭐⭐⭐）

**RDB（快照）**：
- 定时将内存数据快照写入磁盘（.rdb 文件）
- 优：文件紧凑，恢复快；缺：两次快照间数据可能丢失

**AOF（追加日志）**：
- 将每条写命令追加到文件（fsync 策略：always/everysec/no）
- 优：数据更完整；缺：文件大，恢复慢

**混合持久化（推荐）**：RDB + AOF 混合，兼顾速度和数据完整性

### 缓存三大问题（高频⭐⭐⭐⭐⭐）

**缓存穿透**：查询不存在的数据，每次都打到数据库
- 解决：① 缓存空值（TTL 短）② 布隆过滤器拦截

**缓存击穿**：热点 key 过期的瞬间，大量请求并发打到数据库
- 解决：① 互斥锁（只让一个请求重建缓存）② 逻辑过期（不设 TTL，异步更新）

**缓存雪崩**：大量 key 同时过期，或 Redis 宕机
- 解决：① 过期时间加随机值 ② Redis 集群高可用 ③ 降级熔断

### 分布式锁

```
SET lock_key unique_value NX PX 30000
```

**要点**：
- `NX`：不存在才设置（原子性）
- `PX 30000`：30s 过期（防止死锁）
- value 为唯一值（UUID）：删除时验证是自己的锁

**Lua 脚本删除**（原子性）：
```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

**Redisson**：实现了看门狗机制（自动续期），推荐生产使用

---

## 消息队列

### 为什么用 MQ

1. **解耦**：生产者/消费者无需直接依赖
2. **异步**：耗时操作异步处理，提升响应速度
3. **削峰**：流量洪峰时缓冲，保护下游服务

### 消息可靠性保证（高频）

**生产者可靠**：
- RabbitMQ：Confirm 机制 + 持久化
- Kafka：`acks=all`，所有副本确认

**存储可靠**：
- 队列持久化（durable=true）
- 消息持久化（deliveryMode=2）

**消费者可靠**：
- 手动 ACK（处理完再确认）
- 死信队列（失败消息保存）

### 消息幂等性

同一消息被消费多次（网络重试），必须保证业务幂等。
- 方案：唯一消息 ID + 数据库唯一键 / Redis SETNX 去重

### Kafka 核心概念

| 概念 | 解释 |
|------|------|
| Topic | 消息分类 |
| Partition | 分区，提供并行消费能力 |
| Replica | 副本，保证高可用 |
| Consumer Group | 组内消费者各消费部分分区（负载均衡）|
| Offset | 消费位移，由消费者维护（Kafka 0.9+）|

**Kafka 高吞吐量原因**：
1. 顺序写磁盘（比随机写快 100 倍）
2. 零拷贝（`sendfile` 系统调用，减少数据复制）
3. 分区并行
4. 批量发送与压缩
