# MySQL 性能优化实战指南

---

## 目录

- [一、性能优化方法论](#一性能优化方法论)
- [二、SQL 执行流程](#二sql-执行流程)
- [三、索引基础：B+ 树与 InnoDB 存储结构](#三索引基础b-树与-innodb-存储结构)
- [四、索引设计原则](#四索引设计原则)
- [五、索引失效场景与优化对策](#五索引失效场景与优化对策)
- [六、EXPLAIN 执行计划分析](#六explain-执行计划分析)
- [七、SQL 改写与查询优化技巧](#七sql-改写与查询优化技巧)
- [八、InnoDB 与服务器参数调优](#八innodb-与服务器参数调优)
- [九、慢 SQL 排查与监控](#九慢-sql-排查与监控)
- [十、架构层优化](#十架构层优化)
- [十一、面试高频问答](#十一面试高频问答)

---

## 一、性能优化方法论

MySQL 性能优化不是「加索引」这么简单，而是一条有优先级的排查链路。线上问题通常按以下顺序处理：

```
发现慢 SQL → EXPLAIN 分析 → 索引/SQL 改写 → 参数调优 → 架构升级
```

| 层次 | 手段 | 典型收益 | 成本 |
|------|------|----------|------|
| SQL 层 | 索引、改写、避免 SELECT * | 最高，往往 10×~100× | 低 |
| 配置层 | Buffer Pool、连接数、日志 | 中等，20%~50% | 低 |
| 架构层 | 读写分离、分库分表、缓存 | 高，支撑量级扩展 | 高 |

**核心原则**：

1. **先测量再优化**：没有慢查询日志和 EXPLAIN，不要凭感觉加索引。
2. **索引不是越多越好**：每多一个索引，INSERT/UPDATE/DELETE 都要维护 B+ 树，写性能会下降。
3. **尽量让数据在索引层完成**：覆盖索引避免回表，是最常见的低成本优化。
4. **小步验证**：每次只改一处，对比优化前后的 `rows`、`type`、执行时间。

---

## 二、SQL 执行流程

理解 SQL 如何被执行，是定位性能瓶颈的起点。一条查询在 MySQL 中大致经过以下阶段：

```
连接器（认证）→ 查询缓存（8.0 已移除）→ 分析器（词法/语法）
→ 优化器（执行计划）→ 执行器（调用存储引擎）→ 返回结果
```

![MySQL SQL 执行整体架构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-22-mysql-sql-architecture.jpeg)

![SQL 解析与执行过程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/bat-interview/bat-interview-23-sql-parse-execute.jpeg)

| 阶段 | 职责 | 性能相关说明 |
|------|------|--------------|
| 连接器 | 身份验证、权限校验 | 连接数过多会消耗内存；建议使用连接池 |
| 分析器 | 词法分析、语法分析 | 语法错误在此阶段暴露 |
| 优化器 | 选择索引、确定 JOIN 顺序 | **性能瓶颈常出在这里**——选错索引比没索引更糟 |
| 执行器 | 调用存储引擎接口读写数据 | 与 InnoDB 缓冲池命中率密切相关 |
| 存储引擎 | 实际的数据存取 | InnoDB 负责 B+ 树查找、行锁、MVCC |

> **注意**：`EXPLAIN` 模拟的是优化器阶段的执行计划，不会真正执行 SQL（`FROM` 含子查询时除外，子查询仍会被执行并放入临时表）。

---

## 三、索引基础：B+ 树与 InnoDB 存储结构

### 3.1 索引是什么

索引是帮助 MySQL **高效获取数据的排好序的数据结构**。没有索引时，MySQL 只能全表扫描——每读一行就是一次磁盘 I/O，10 万行数据意味着 10 万次 I/O，代价极高。

MySQL 以 **磁盘 I/O 次数** 衡量查询效率，因此选用 **B+ 树** 作为 InnoDB 默认索引结构。

### 3.2 为什么选 B+ 树

| 数据结构 | 问题 |
|----------|------|
| 二叉树 | 有序插入退化为链表，深度过大 |
| 红黑树 / AVL | 树深度大，磁盘 I/O 频繁；适合内存，不适合磁盘 |
| B 树 | 非叶子节点也存数据，单页容纳关键字少 |
| Hash | 只支持等值查询，不支持范围查询和排序 |
| **B+ 树** | 非叶子节点仅存索引，叶子节点有序链表，I/O 次数少，支持范围扫描 |

**B+ 树 vs B 树的关键差异**：

- B+ 树所有数据都在叶子节点，非叶子节点只做索引（稀疏索引）
- 叶子节点通过双向链表连接，**区间查询**效率极高
- 单次磁盘 I/O 能加载更多索引项，树高更低（通常 3~4 层即可支撑千万级数据）

![InnoDB B+ 树存储结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/java-interview/java-interview-129-innodb.png)

### 3.3 B+ 树节点容量

InnoDB 以 **页（Page）** 为最小存储单位，默认 **16 KB**。

- 节点大小 ≈ 1 页时最优：读一次 I/O 恰好加载一个节点
- 若节点 < 1 页，浪费空间；若 > 1 页，一次 I/O 需读多页，同样浪费

![B+ 树节点示意](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202411170256341.png)

### 3.4 聚集索引与非聚集索引

| 类型 | 说明 | InnoDB 实现 |
|------|------|-------------|
| **聚集索引（聚簇索引）** | 索引与数据存在同一 B+ 树，找到索引即找到数据 | 主键索引；叶子节点存完整行记录 |
| **非聚集索引（二级索引）** | 索引与数据分开，叶子节点存主键值 | 普通索引；查非索引列需**回表** |

**InnoDB 必须有聚集索引**，选择规则：

1. 显式定义主键
2. 否则选第一个 NOT NULL 的唯一索引
3. 都没有则自动生成 6 字节 ROWID

**为什么推荐自增整型主键**：

- 自增主键顺序插入，页分裂少，写入局部性好
- 随机主键（如 UUID）导致频繁页分裂和数据移动，碎片多
- 整型比较比 UUID 字符串快，占用空间更小

**为什么二级索引叶子节点只存主键**：

- 数据已在聚集索引的 B+ 树上，二级索引再存一份数据浪费空间
- DML 时只需更新二级索引中的主键引用，保持一致性

### 3.5 联合索引的底层结构

联合索引 `(name, age, position)` 在 B+ 树中按 **先 name、再 age、再 position** 的字典序排列：

![联合索引底层结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/202411170257683.png)

### 3.6 覆盖索引

覆盖索引指查询所需的列**全部在二级索引中**，无需回表到聚集索引取数据。

```sql
-- 索引 idx_name_age(name, age)
SELECT name, age FROM employees WHERE name = 'LiLei';
-- Extra: Using index → 覆盖索引，性能最优
```

### 3.7 存储引擎选型

| 引擎 | 索引结构 | 事务 | 行锁 | 适用场景 |
|------|----------|------|------|----------|
| **InnoDB** | B+ 树 | 支持 | 支持 | 默认选择，OLTP、高并发更新 |
| MyISAM | B+ 树 | 不支持 | 表锁 | 只读、读多写少（已不推荐） |
| Memory | Hash / B 树 | 不支持 | 表锁 | 临时表、会话级缓存 |

InnoDB 适用场景：频繁更新、需要事务、需要崩溃恢复、需要外键约束。

![索引数据结构对比](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/java-interview/java-interview-130-index.png)

---

## 四、索引设计原则

![常见索引设计原则](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/java-interview/java-interview-131-diagram.png)

### 4.1 基本原则

1. **为常作为查询条件的列建索引**（WHERE、JOIN ON、ORDER BY、GROUP BY）
2. **选择区分度高的列**：区分度 = `COUNT(DISTINCT col) / COUNT(*)`，建议 > 0.1
3. **优先使用唯一索引**：唯一值定位更快
4. **尽量扩展联合索引，不要频繁新建单列索引**
5. **限制索引数量**：索引越多，写操作越慢
6. **索引列保持「干净」**：不参与计算、不使用函数、避免隐式类型转换
7. **尽量使用前缀索引**：对长字符串列 `INDEX idx(col(20))`
8. **删除不再使用的索引**

### 4.2 最左前缀匹配原则

联合索引 `(a, b, c)` 相当于建立了 `(a)`、`(a,b)`、`(a,b,c)` 三个索引的前缀。

| SQL 条件 | 是否使用索引 |
|----------|--------------|
| `WHERE a = 1` | ✅ 使用 a |
| `WHERE a = 1 AND b = 2` | ✅ 使用 a, b |
| `WHERE a = 1 AND b = 2 AND c = 3` | ✅ 使用 a, b, c |
| `WHERE b = 2` | ❌ 跳过 a，索引失效 |
| `WHERE a = 1 AND c = 3` | ⚠️ 只用到 a，c 无法使用 |

**范围查询截断后续列**：`WHERE a = 1 AND b > 2 AND c = 3` 中，`c` 无法使用索引，因为 `b` 做了范围扫描后，后续列在索引树中不再有序。

### 4.3 key_len 计算规则

`key_len` 表示索引使用的字节数，可判断联合索引实际命中了哪些列：

| 类型 | 计算方式 |
|------|----------|
| `int` | 4 字节 |
| `bigint` | 8 字节 |
| `datetime` | 8 字节 |
| `timestamp` | 4 字节 |
| `char(n)` utf8 | 3n 字节 |
| `varchar(n)` utf8 | 3n + 2 字节（+2 存长度） |
| 允许 NULL | 额外 +1 字节 |

示例：索引 `(name, age, position)`，`name varchar(24)`、`age int`、`position varchar(20)`：

```
key_len = (3×24+2) + 4 + (3×20+2) = 74 + 4 + 62 = 140
```

---

## 五、索引失效场景与优化对策

### 5.1 常见失效场景

| 场景 | 示例 | 原因 |
|------|------|------|
| 对索引列做运算 | `WHERE YEAR(hire_time) = 2024` | 索引树存的是原始值，无法匹配 |
| 隐式类型转换 | `WHERE phone = 13800138000`（phone 是 varchar） | MySQL 对列做 CAST，索引失效 |
| 左模糊 LIKE | `WHERE name LIKE '%Li'` | 无法利用索引有序性 |
| 不等于 | `WHERE status != 1` | 优化器可能选择全表扫描 |
| OR 连接不同列 | `WHERE a = 1 OR c = 3` | 无法同时走同一联合索引 |
| 范围查询后的列 | `WHERE a=1 AND b>2 AND c=3` | b 范围扫描后 c 无序 |
| IS NULL / IS NOT NULL | 视数据分布而定 | 优化器可能放弃索引 |
| 数据量过小 | 表仅几百行 | 全表扫描比走索引更快 |

### 5.2 LIKE 优化

| 模式 | 能否走索引 | 说明 |
|------|------------|------|
| `LIKE 'abc%'` | ✅ | 前缀匹配，等价于范围查询 |
| `LIKE '%abc'` | ❌ | 后缀模糊，全表扫描 |
| `LIKE '%abc%'` | ❌ | 中间模糊，全表扫描 |

**`%abc%` 的替代方案**：

1. 使用覆盖索引减少回表开销
2. 引入 Elasticsearch 等全文搜索引擎
3. 反向存储 + 前缀索引（特定场景）

### 5.3 OR 条件优化

`WHERE a = 1 AND b = 2 OR c = 3` 在 `c` 无法利用 `(a,b,c)` 索引时，优化器可能全表扫描。

**改写为 UNION**：

```sql
(SELECT * FROM my_table WHERE a = 1 AND b = 2)
UNION
(SELECT * FROM my_table WHERE c = 3);
```

### 5.4 范围查询优化

当单次范围扫描数据量过大，优化器可能放弃索引。可将大区间拆成多个小范围：

```sql
-- 原 SQL：扫描 100 万行
SELECT * FROM orders WHERE create_time BETWEEN '2024-01-01' AND '2024-12-31';

-- 优化：按月分批
SELECT * FROM orders WHERE create_time BETWEEN '2024-01-01' AND '2024-01-31'
UNION ALL
SELECT * FROM orders WHERE create_time BETWEEN '2024-02-01' AND '2024-02-29';
-- ...
```

### 5.5 索引使用决策速查

```
等值查询（=）         → ref / const，优先走索引
范围查询（> < BETWEEN）→ range，注意截断后续列
排序（ORDER BY）      → 索引列顺序与 ORDER BY 一致可免 filesort
分组（GROUP BY）      → 同上
LIKE 'xx%'           → range
LIKE '%xx'           → ALL（全表扫描）
```

---

## 六、EXPLAIN 执行计划分析

`EXPLAIN` 模拟优化器执行 SQL，返回执行计划而非真实结果，是 SQL 调优的核心工具。

### 6.1 5 秒法则：只看 4 列

```
一看 type → 访问类型
二看 key  → 实际使用的索引
三看 rows → 预估扫描行数
四看 Extra → 额外操作
```

| type | 含义 | 性能 |
|------|------|------|
| `system` | 表仅一行 | ⭐⭐⭐⭐⭐ |
| `const` | 主键/唯一索引等值 | ⭐⭐⭐⭐⭐ |
| `eq_ref` | JOIN 时唯一索引匹配 | ⭐⭐⭐⭐ |
| `ref` | 普通索引等值 | ⭐⭐⭐ |
| `range` | 范围扫描 | ⭐⭐⭐ |
| `index` | 全索引扫描 | ⭐⭐ |
| `ALL` | 全表扫描 | ❌ |

**目标**：至少 `range`，最好 `ref` / `const`；出现 `ALL` 基本需要优化。

**rows 经验值**：

| rows | 评估 |
|------|------|
| < 100 | 很好 |
| < 1 000 | 可以 |
| < 10 000 | 一般 |
| > 100 000 | 可能有问题 |

注意：`rows × 循环次数` 才是实际扫描量（如 Nested Loop JOIN 中内表 rows=100、循环 1000 次 = 10 万行）。

### 6.2 各列详解

#### id

`SELECT` 的序列号，有几个 `SELECT` 就有几个 id。id 越大越先执行；id 相同则从上到下执行。

#### select_type

| 值 | 含义 |
|----|------|
| `SIMPLE` | 简单查询，无子查询和 UNION |
| `PRIMARY` | 最外层查询 |
| `SUBQUERY` | SELECT 中的子查询 |
| `DERIVED` | FROM 中的子查询（派生表） |
| `UNION` | UNION 中第二个及之后的 SELECT |

#### type（见上表）

#### possible_keys / key

- `possible_keys`：可能使用的索引
- `key`：实际使用的索引；`NULL` 表示未走索引

**高频面试点**：`possible_keys` 有值但 `key = NULL`，说明索引存在但优化器认为全表扫描更划算（数据量小、区分度低、范围过大等）。

#### Extra

| 值 | 含义 | 优化建议 |
|----|------|----------|
| `Using index` | 覆盖索引 | ✅ 好 |
| `Using where` | 存储引擎返回后在 Server 层过滤 | 检查是否索引覆盖不足 |
| `Using index condition` | 索引下推（ICP） | 一般可接受 |
| `Using temporary` | 使用临时表 | ⚠️ 考虑加索引或改写 SQL |
| `Using filesort` | 额外排序 | ⚠️ 让 ORDER BY 走索引 |
| `Select tables optimized away` | 聚合函数直接走索引 | ✅ 好 |

**Using temporary 示例**：

```sql
-- 无索引：需要临时表去重
EXPLAIN SELECT DISTINCT name FROM actor;

-- 有 name 索引：Extra = Using index
EXPLAIN SELECT DISTINCT name FROM film;
```

**Using filesort 示例**：

```sql
-- 无索引：扫描全表 + 排序
EXPLAIN SELECT * FROM actor ORDER BY name;

-- 有 idx_name：Extra = Using index
EXPLAIN SELECT * FROM film ORDER BY name;
```

### 6.3 EXPLAIN 变种

```sql
-- 额外优化信息
EXPLAIN EXTENDED SELECT * FROM actor;
SHOW WARNINGS;

-- 分区表显示访问的分区
EXPLAIN PARTITIONS SELECT * FROM partitioned_table WHERE id = 1;
```

MySQL 8.0+ 推荐使用 `EXPLAIN ANALYZE`，可获取真实执行时间和循环次数：

```sql
EXPLAIN ANALYZE SELECT * FROM employees WHERE name = 'LiLei';
```

### 6.4 实战案例

**慢 SQL**：

```
type: ALL
key:  NULL
rows: 100000
Extra: Using where
```

5 秒判断：全表扫描 + 无索引 + 10 万行 + Server 层过滤 → **需要加索引**。

**好 SQL**：

```
type: ref
key:  idx_user_id
rows: 10
Extra: Using index
```

5 秒判断：走索引 + 扫描 10 行 + 覆盖索引 → **优秀**。

---

## 七、SQL 改写与查询优化技巧

### 7.1 减少 SELECT *

```sql
-- ❌ 回表取所有列
SELECT * FROM orders WHERE user_id = 100;

-- ✅ 只查需要的列，可能走覆盖索引
SELECT id, status, create_time FROM orders WHERE user_id = 100;
```

### 7.2 避免在 WHERE 中对列运算

```sql
-- ❌ 索引失效
SELECT * FROM user WHERE DATE(create_time) = '2024-06-01';

-- ✅ 改写为范围
SELECT * FROM user
WHERE create_time >= '2024-06-01 00:00:00'
  AND create_time <  '2024-06-02 00:00:00';
```

### 7.3 小表驱动大表

JOIN 时让 **小结果集作为驱动表**，减少循环次数：

```sql
-- 假设 user 表 100 行，order 表 100 万行
-- ✅ user 驱动 order
SELECT * FROM user u JOIN order o ON u.id = o.user_id WHERE u.id = 1;
```

MySQL 优化器通常会自动选择，但复杂 JOIN 可用 `STRAIGHT_JOIN` 强制顺序。

### 7.4 深分页优化

```sql
-- ❌ OFFSET 越大越慢，需扫描并丢弃前 N 行
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- ✅ 延迟关联：先走索引取 id，再 JOIN
SELECT o.* FROM orders o
INNER JOIN (
  SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
) t ON o.id = t.id;

-- ✅ 记住上次最大 id（无跳页需求时）
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

### 7.5 COUNT 优化

| 方式 | 速度 | 说明 |
|------|------|------|
| `COUNT(*)` | 快 | InnoDB 会选最小的二级索引统计 |
| `COUNT(主键)` | 较快 | 走聚集索引 |
| `COUNT(列)` | 慢 | 需判断 NULL |
| 缓存 / 冗余字段 | 最快 | 业务允许近似值时使用 |

### 7.6 批量写入优化

```sql
-- ❌ 逐条 INSERT
INSERT INTO log VALUES (...);
INSERT INTO log VALUES (...);

-- ✅ 批量 INSERT
INSERT INTO log VALUES (...), (...), (...);

-- ✅ 关闭 autocommit，批量提交
SET autocommit = 0;
-- 批量 INSERT ...
COMMIT;
```

### 7.7 存储过程优化思路

1. 用集合操作（聚合函数）替代游标循环
2. 中间结果放临时表并加索引
3. 事务尽量短，减少锁持有时间
4. 查找语句不要放在循环内

---

## 八、InnoDB 与服务器参数调优

### 8.1 核心参数

| 参数 | 建议 | 说明 |
|------|------|------|
| `innodb_buffer_pool_size` | 物理内存的 60%~70% | 最重要的参数，缓存数据和索引 |
| `innodb_buffer_pool_instances` | 8（大内存时） | 减少缓冲池锁竞争 |
| `innodb_log_file_size` | 256M~1G | redo log 大小，影响写入性能 |
| `innodb_flush_log_at_trx_commit` | 1（默认，最安全）/ 2（性能优先） | 1=每次提交刷盘 |
| `sync_binlog` | 1（最安全）/ 0（性能优先） | binlog 刷盘策略 |
| `max_connections` | 按业务评估，通常 500~2000 | 过大浪费内存 |
| `thread_cache_size` | 32~64 | 复用线程，减少创建开销 |
| `table_open_cache` | 2000+ | 表缓存，避免频繁打开表 |

### 8.2 Buffer Pool 命中率

```sql
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
```

```
命中率 = 1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)
```

目标：**> 99%**。过低说明内存不足或存在大量全表扫描。

### 8.3 连接池配置（应用层）

应用侧应使用连接池（HikariCP、Druid 等），避免频繁创建连接：

```properties
# HikariCP 示例
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
```

经验值：`connections = ((core_count × 2) + effective_spindle_count)`，通常 10~20 足够，不是越大越好。

### 8.4 事务与锁对性能的影响

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|----------|------|------------|------|------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 最高 |
| READ COMMITTED | 不可能 | 可能 | 可能 | 较高 |
| REPEATABLE READ（MySQL 默认） | 不可能 | 不可能 | 可能* | 中等 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 | 最低 |

*InnoDB 的 RR 通过 MVCC + Next-Key Lock 在很大程度上避免幻读。

**性能建议**：

- 事务尽量短，减少行锁持有时间
- 避免 `SELECT ... FOR UPDATE` 锁不必要的数据
- 更新同一热点行（如库存、计数器）是常见瓶颈，考虑分段锁或 Redis 预减

### 8.5 表设计建议

1. **遵循三范式**，但适度反范式化（冗余换查询性能）
2. **字段类型尽量小**：`TINYINT` 优于 `INT`，`VARCHAR(50)` 优于 `VARCHAR(255)`
3. **避免 NULL**：用默认值替代，减少索引判断复杂度
4. **大字段拆分**：TEXT/BLOB 拆到扩展表，主表保持精简
5. **定期清理碎片**：`OPTIMIZE TABLE`（会锁表，低峰执行）

---

## 九、慢 SQL 排查与监控

### 9.1 开启慢查询日志

```sql
-- 查看当前配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志（动态，重启失效）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;          -- 超过 1 秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未走索引的 SQL
```

`my.cnf` 持久化配置：

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

### 9.2 分析慢查询日志

```bash
# mysqldumpslow：MySQL 自带
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# pt-query-digest：Percona 工具，功能更强
pt-query-digest /var/log/mysql/slow.log
```

### 9.3 常用诊断命令

```sql
-- 查看当前正在执行的 SQL
SHOW FULL PROCESSLIST;

-- 查看 InnoDB 状态（锁等待、事务）
SHOW ENGINE INNODB STATUS\G

-- 查看表索引使用情况
SHOW INDEX FROM table_name;

-- 查看表统计信息
ANALYZE TABLE table_name;
```

### 9.4 优化闭环

```
1. 慢查询日志 / APM 发现慢 SQL
2. EXPLAIN / EXPLAIN ANALYZE 分析执行计划
3. 加索引 / 改写 SQL / 调整参数
4. 对比优化前后 rows、耗时
5. 上线后持续监控
```

---

## 十、架构层优化

当单库单表优化到极限（通常单表 > 500 万~1000 万行，或 QPS > 5000），需要考虑架构升级。

### 10.1 读写分离

- 主库写，从库读
- 注意主从延迟导致读到旧数据
- 强一致读走主库，普通查询走从库

### 10.2 分库分表

| 方式 | 说明 | 场景 |
|------|------|------|
| 垂直拆分 | 按业务模块拆库（用户库、订单库） | 业务解耦 |
| 水平拆分 | 按规则（user_id 哈希）拆表 | 单表数据量过大 |

常见中间件：ShardingSphere、MyCat。

### 10.3 缓存

```
读：先查 Redis → 未命中查 MySQL → 写入 Redis
写：先更新 MySQL → 删除/更新 Redis 缓存
```

注意：缓存穿透、击穿、雪崩的防护。

### 10.4 Binlog 与主从

| 格式 | 优点 | 缺点 |
|------|------|------|
| STATEMENT | 日志量小 | 可能主从不一致（UDF、sleep） |
| ROW | 精确 | 日志量大 |
| MIXED | 折中 | 规则复杂 |

---

## 十一、面试高频问答

### Q1：一条 SQL 慢，你怎么排查？

> 先看慢查询日志定位 SQL → `EXPLAIN` 看 type/key/rows/Extra → 判断是否全表扫描、是否回表、是否有 filesort/temporary → 针对性加索引或改写 SQL → 检查 Buffer Pool 命中率和锁等待 → 必要时架构升级。

### Q2：什么时候不适合加索引？

> 1）表数据量很小；2）写多读少的表；3）区分度很低的列（如性别）；4）频繁更新的列；5）索引已经够用，再加收益递减。

### Q3：覆盖索引是什么？有什么好处？

> 查询列全部在二级索引中，不需要回表到聚集索引。好处：减少 I/O、避免回表随机读、Extra 显示 Using index。

### Q4：最左前缀原则是什么？

> 联合索引 `(a,b,c)` 必须从最左列开始连续匹配。跳过 a 直接用 b 无法走索引；a 等值 + b 范围后 c 无法使用索引。

### Q5：EXPLAIN 中 type=ALL 怎么办？

> 全表扫描，检查 WHERE 条件是否能走索引、是否存在索引失效（函数、类型转换、左模糊）、数据量是否过大需要分页优化或分表。

### Q6：InnoDB 为什么用 B+ 树而不是 B 树？

> B+ 树非叶子节点不存数据，单页容纳更多索引项，树高更低，I/O 更少；叶子节点有序链表，区间查询高效；适合 MySQL 这种磁盘型数据库。

### Q7：为什么推荐自增主键？

> 顺序插入，页分裂少，写入性能好；整型占用空间小、比较快；二级索引只需存主键，聚集索引紧凑。

### Q8：如何优化深分页？

> 避免大 OFFSET；用延迟关联（子查询先取 id 再 JOIN）或记住上次最大 id 的方式分页。

### Q9：索引下推（ICP）是什么？

> MySQL 5.6+ 特性。联合索引 `(name, age)` 查询 `WHERE name LIKE 'Li%' AND age = 20` 时，存储引擎在索引层直接过滤 age，减少回表行数。Extra 显示 `Using index condition`。

### Q10：乐观锁和悲观锁怎么选？

> 读多写少用乐观锁（版本号/CAS）；写多读少、竞争激运用悲观锁（`SELECT ... FOR UPDATE`）。

```sql
-- 乐观锁
UPDATE account SET balance = balance - 100, version = version + 1
WHERE id = 1 AND version = 5;

-- 悲观锁
SELECT * FROM account WHERE id = 1 FOR UPDATE;
```

---

## 附录：索引优化实战 DDL

```sql
CREATE TABLE `employees` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(24) NOT NULL DEFAULT '' COMMENT '姓名',
  `age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄',
  `position` varchar(20) NOT NULL DEFAULT '' COMMENT '职位',
  `hire_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间',
  PRIMARY KEY (`id`),
  KEY `idx_name_age_position` (`name`,`age`,`position`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='员工记录表';
```

```sql
-- 验证最左前缀
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei';
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' AND age = 22;
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' AND age = 22 AND position = 'manager';

-- 验证范围截断
EXPLAIN SELECT * FROM employees WHERE name = 'LiLei' AND age > 22 AND position = 'manager';
```
