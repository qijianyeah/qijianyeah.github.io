# MySQL 索引篇

---

## 目录

- [一、一条 Select 语句的执行流程](#一条-select-语句的执行流程)
- [二、索引介绍](#二索引介绍)
- [三、索引的使用](#三索引的使用)
- [四、索引的数据结构](#四索引的数据结构)
- [五、MySQL 索引实现](#五mysql-索引实现)
- [六、索引创建原则](#六索引创建原则)
- [附：索引失效分析](#附索引失效分析)

---

## 一、一条 Select 语句的执行流程

以如下查询为例：

```sql
SELECT * FROM city WHERE city_id = 1;
```

![Select 语句执行流程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-001-select-execution-flow.png)

### 1.1 MySQL Server 层

| 步骤 | 组件 | 作用 |
|------|------|------|
| 1 | 连接器 | 身份认证、权限校验、维持连接 |
| 2 | 分析器 | 词法分析、语法分析，生成语法树 |
| 3 | 优化器 | 选择索引、确定 JOIN 顺序、生成执行计划 |
| 4 | 执行器 | 调用存储引擎接口读写数据 |
| 5 | 返回结果 | 将记录集返回客户端 |

> MySQL 8.0 已移除查询缓存，分析器之后直接进入优化器。

### 1.2 InnoDB 存储引擎层

执行器调用存储引擎后，InnoDB 根据优化器决策：

1. **不走索引** → 全表扫描（遍历聚簇索引叶子节点）。
2. **走主键索引** → 直接在聚簇索引 B+ 树上定位行记录（无需回表）。
3. **走辅助索引** → 先在二级索引找到主键 id → 再回聚簇索引取完整行（回表）。

理解这条链路，是后续分析「为什么加索引能快」「为什么会回表」的基础。

---

## 二、索引介绍

### 2.1 索引是什么

官方定义：索引是帮助 MySQL **高效获取数据**的数据结构。

通俗理解：索引像书的目录，通过目录可以快速定位正文页码，而不必从第一页翻到最后。

要点：

- 索引本身也占磁盘空间，不可能全部常驻内存。
- 索引通常以 **B+ 树** 组织（多路平衡查找树，不一定是二叉）。
- 常见类型：聚簇索引、辅助索引、覆盖索引、组合索引、前缀索引、唯一索引等。

### 2.2 索引的优势和劣势

**优势**：

| 优势 | 说明 |
|------|------|
| 降低 IO | 减少全表扫描，快速定位数据页 |
| 降低排序成本 | 索引列天然有序，利于 `ORDER BY` |
| 加速排序分组 | 组合索引按列顺序排列，可优化 `ORDER BY` / `GROUP BY` |

**劣势**：

| 劣势 | 说明 |
|------|------|
| 占空间 | 索引树需要额外磁盘与内存 |
| 降低写性能 | INSERT / UPDATE / DELETE 需同步维护索引 |

---

## 三、索引的使用

### 3.1 索引的类型

#### 按功能分类

| 类型 | 说明 |
|------|------|
| 主键索引 | 值唯一且非空，InnoDB 中即聚簇索引 |
| 普通索引 | 无唯一性限制，允许重复值和 NULL |
| 唯一索引 | 值唯一，允许 NULL（NULL 可有多行） |
| 全文索引 | 用于 `CHAR` / `VARCHAR` / `TEXT` 的全文检索 |
| 空间索引 | 5.7+ 支持，用于 OpenGIS 几何数据 |
| 前缀索引 | 对文本列指定前缀长度建索引，节省空间 |

#### 按列数分类

| 类型 | 说明 |
|------|------|
| 单列索引 | 一个索引只含一列 |
| 组合索引 | 多列联合索引，遵循最左前缀原则 |

> 业务中优先使用组合索引（主键除外），一棵组合索引 `(a,b,c)` 等价于 `(a)`、`(a,b)`、`(a,b,c)` 三棵索引的效果。

#### 常用 DDL

```sql
-- 主键索引
ALTER TABLE table_name ADD PRIMARY KEY (column_name);

-- 普通索引
ALTER TABLE table_name ADD INDEX index_name (column_name);

-- 唯一索引
CREATE UNIQUE INDEX index_name ON table(column_name);

-- 前缀索引
ALTER TABLE table_name ADD INDEX index_name (column1(10));

-- 组合索引
ALTER TABLE table_name ADD INDEX index_name (column1, column2);

-- 全文索引
CREATE TABLE t_fulltext (
  id INT NOT NULL AUTO_INCREMENT,
  content VARCHAR(100),
  PRIMARY KEY (id),
  FULLTEXT KEY idx_content (content)
) ENGINE=InnoDB;

SELECT * FROM t_fulltext WHERE MATCH(content) AGAINST('关键词');
```

全文索引适合数据量小、并发低的场景；生产环境大规模检索通常用 Elasticsearch 等专业方案。

### 3.2 删除索引

```sql
DROP INDEX index_name ON table_name;
```

### 3.3 查看索引

```sql
SHOW INDEX FROM table_name\G
```

---

## 四、索引的数据结构

### 4.1 索引的要求

索引数据结构至少要高效支持：

1. **等值查询**：`WHERE age = 76`
2. **范围查询**：`WHERE age >= 76 AND age <= 86`

同时兼顾 **查询时间** 与 **存储空间**。

### 4.2 数据结构的选用

常见候选：Hash 表、二叉查找树、平衡二叉树（红黑树）、B 树、B+ 树。

可视化学习推荐：[Data Structure Visualizations](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

### 4.3 Hash 表

Java 的 `HashMap` 即 Hash 表结构，Key 存索引列，Value 存行记录或磁盘地址。

- 等值查询：O(1)，极快。
- 范围查询：不支持，只能全表扫描。
- InnoDB 有**自适应 Hash 索引**，但用户无法手动创建 Hash 索引。

### 4.4 二叉查找树

每个节点最多两个子节点，左小右大。

![二叉查找树](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-002-binary-search-tree.png)

- 有序数据（如 age）时，查询效率较好。
- 若按自增 id 插入，树会退化为链表，查询退化为 O(n)。

### 4.5 平衡二叉查找树

通过左旋/右旋保持左右子树高度差不超过 1，查询复杂度 O(log₂n)。

![平衡二叉查找树](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-003-balanced-binary-tree.png)

**问题**：

1. 树高 = 磁盘 IO 次数。100 万数据约需 20 次 IO（每次寻道约 10ms → 200ms）。
2. 不支持高效范围查询，需多次中序遍历。

### 4.6 B 树：改造二叉树

核心思路：**每个节点存储多个键值**，降低树高，提高一次 IO 的有效数据量。

假设每个键 8 字节 + 两个指针各 4 字节 = 16 字节/索引项，InnoDB 页 16KB 可存约 1000 个索引项。100 万数据只需 2 层（1000 × 1000）。

![B 树结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-004-b-tree-structure.png)

**B 树特点**：

1. 多叉平衡树，非叶子节点也有数据。
2. 节点内键值有序。
3. 父节点键值不出现在子节点。
4. 所有叶子在同一层，叶子间无指针连接。

**等值查询示例**（查找 15）：磁盘块 1 → 2 → 7，3 次 IO。

**缺点**：

1. 范围查询需反复从根节点遍历。
2. 若节点存完整行记录，单页容纳索引项变少，树变高，IO 增多。

### 4.7 B+ 树：MySQL 的选择

B+ 树在 B 树基础上的关键改造：

| 对比 | B 树 | B+ 树 |
|------|------|-------|
| 数据存储位置 | 非叶子节点和叶子节点都存 | **仅叶子节点存数据** |
| 非叶子节点 | 存键值+数据 | 只存键值（索引项） |
| 叶子节点连接 | 无 | **双向链表**连接 |
| 等值命中内节点 | 可中途返回 | 必须到叶子节点 |

![B+ 树结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-005-b-plus-tree.png)

**等值查询**（查找 15）：磁盘块 1 → 2 → 5，3 次 IO，在叶子节点取数据。

**范围查询**（15 ~ 26）：

1. 先定位到 15 所在叶子节点（3 次 IO）。
2. 沿叶子链表向后扫描，无需回到根节点。
3. 直到超过 26 停止。

这正是 MySQL（InnoDB / MyISAM）选择 B+ 树的原因：**等值 + 范围查询都高效**。

---

## 五、MySQL 索引实现

### 5.1 MyISAM 索引

MyISAM 的**索引文件**（`.MYI`）与**数据文件**（`.MYD`）分离。B+ 树叶子节点存的是**行记录的磁盘地址**，而非完整行数据。

```sql
CREATE TABLE t_user_myisam (
  id INT NOT NULL AUTO_INCREMENT,
  username VARCHAR(20),
  age INT,
  PRIMARY KEY (id) USING BTREE,
  KEY idx_age (age) USING BTREE
) ENGINE=MyISAM;
```

![MyISAM 索引结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-006-myisam-index.png)

#### 主键索引等值查询

```sql
SELECT * FROM t_user_myisam WHERE id = 30;
```

1. 主键 B+ 树定位 id=30（3 次 IO）。
2. 从叶子节点取磁盘地址。
3. 到 `.MYD` 数据文件读取完整行（1 次 IO）。

**共 4 次磁盘 IO**。

#### 主键索引范围查询

```sql
SELECT * FROM t_user_myisam WHERE id BETWEEN 30 AND 49;
```

定位起始叶子节点后，沿链表向后遍历，每条记录各需 1 次数据文件 IO。

#### 辅助索引

辅助索引与主键索引结构相同，叶子节点存**磁盘地址**。因键值可重复，等值查询也按范围方式在索引树中扫描。

### 5.2 InnoDB 索引

#### 聚簇索引规则

InnoDB 表必有一个聚簇索引，叶子节点存储**完整行记录**：

1. 有主键 → 主键即聚簇索引。
2. 无主键 → 选第一个非 NULL 唯一索引。
3. 都没有 → 隐式 6 字节 `ROWID` 作为聚簇索引。

辅助索引叶子节点只存**主键值**，查完整行需回表。

```sql
CREATE TABLE t_user_innodb (
  id INT NOT NULL AUTO_INCREMENT,
  username VARCHAR(20),
  age INT,
  PRIMARY KEY (id) USING BTREE,
  KEY idx_age (age) USING BTREE
) ENGINE=InnoDB;
```

#### 主键索引（聚簇索引）

![InnoDB 聚簇索引](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-007-innodb-primary-index.png)

```sql
SELECT * FROM t_user_innodb WHERE id = 30;
```

在聚簇索引叶子节点直接拿到完整行，**3 次 IO**，无需再访问数据文件。

```sql
SELECT * FROM t_user_innodb WHERE id BETWEEN 30 AND 49;
```

定位后沿叶子链表扫描即可，**2 + 叶子节点数** 次 IO，比 MyISAM 少一次数据文件寻址。

#### 辅助索引与回表

![InnoDB 辅助索引](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-008-innodb-secondary-index.png)

叶子节点按 `(age, id)` 排序：先按 age，age 相同再按 id。

```sql
SELECT * FROM t_user_innodb WHERE age = 22;
```

1. 在 age 索引树定位 age=22 的条目（3 次 IO）。
2. 取主键 id=18、id=49，分别回聚簇索引查完整行（各 3 次 IO）。

**回表**：辅助索引找到主键后，再到聚簇索引取行数据的过程。

> `SELECT *` 几乎必然回表；只查索引列可用覆盖索引避免回表。

### 5.3 组合索引

#### 存储结构

```sql
CREATE TABLE t_multiple_index (
  id INT NOT NULL AUTO_INCREMENT,
  a INT, b INT, c VARCHAR(10), d VARCHAR(10),
  PRIMARY KEY (id),
  KEY idx_abc (a, b, c)
) ENGINE=InnoDB;
```

![组合索引存储结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-009-composite-index-structure.png)

B+ 树按 `(a, b, c, id)` 排序：a 相同比 b，b 相同比 c，三列都相同比主键 id。最底层不存在完全相同的索引项。

#### 查找方式

```sql
SELECT * FROM t_multiple_index WHERE a = 13 AND b = 16 AND c = 4;
```

![组合索引查找过程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-010-composite-index-lookup.png)

1. 根节点比较 a=13，走左路。
2. 内节点比较 a=13、b=14，继续向下。
3. 叶子节点找到 `(13,16,4,id=1)`，回表取行。

#### 最左前缀匹配原则

![最左前缀原则](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-011-leftmost-prefix.png)

B+ 树先按 a 全局有序，b 只在 a 相等的小范围内有序，c 只在 a、b 都相等时有序。没有 a 条件，优化器无法确定从哪个节点开始检索。

**能用索引 `idx_abc(a,b,c)` 的查询**：

```sql
SELECT * FROM t_multiple_index WHERE a = 13;
SELECT * FROM t_multiple_index WHERE a = 13 AND b = 16;
SELECT * FROM t_multiple_index WHERE a = 13 AND b = 16 AND c = 4;
SELECT * FROM t_multiple_index WHERE a = 13 AND b > 13;      -- b 范围，c 失效
SELECT * FROM t_multiple_index WHERE a > 11 AND b = 14;      -- 仅 a 走索引
SELECT * FROM t_multiple_index WHERE a = 16 AND c = 4;       -- 仅 a 走索引
```

**不能用组合索引的查询**：

```sql
SELECT * FROM t_multiple_index WHERE b = 16 AND c = 4;  -- 缺少 a
SELECT * FROM t_multiple_index WHERE c = 4;            -- 缺少 a
```

**WHERE 条件顺序无关**，优化器会重排：

```sql
-- 等价于 WHERE a = 13 AND b = 16 AND c = 4
SELECT * FROM t_multiple_index WHERE b = 16 AND c = 4 AND a = 13;
```

**范围查询截断规则**：遇到 `>`、`<`、`BETWEEN`、`LIKE 'xx%'` 等范围条件后，**右侧列**无法继续利用索引排序定位（但覆盖索引场景除外）。

**索引使用口诀**：

> 全值匹配我最爱，最左前缀要遵守；  
> 带头大哥不能死，中间兄弟不能断；  
> 索引列上不计算，范围之后全失效；  
> Like 百分写最右，覆盖索引不写星；  
> 不等空值还有 OR，索引失效要少用。

#### 组合索引创建原则

1. 频繁出现在 `WHERE` 的列优先。
2. `ORDER BY` / `GROUP BY` 的列按使用顺序编入索引（`ORDER BY a, b` 需要 `(a, b)` 而非 `(b, a)`）。
3. 频繁出现在 `SELECT` 的列编入索引，利于覆盖索引。

列顺序：把**区分度高、使用频繁**的列放前面。

**面试题**：`SELECT * FROM t WHERE a = 1 AND b > 2 ORDER BY c`，除了 `(a, b)` 还能怎么建？

可考虑 `(a, c)`：a 等值定位后，c 在索引中已有序，可避免 filesort。具体选 `(a,b)` 还是 `(a,c)` 取决于 b、c 的区分度与数据分布，需用 `EXPLAIN` 验证。

### 5.4 覆盖索引

辅助索引叶子节点包含 `(a, b, c, id)`，若查询列全部在索引中，无需回表：

```sql
SELECT a, b, c FROM t_multiple_index WHERE a = 13 AND b = 16;
-- Extra: Using index
```

```sql
SELECT a, b FROM t_multiple_index WHERE b = 16;
```

虽不满足最左前缀，但所需列都在索引中，优化器可能选择**全索引扫描**（`index` 类型），仍比回表查聚簇索引更省 IO。

覆盖索引优势：

- 减少回表，降低磁盘 IO。
- 索引项比完整行小，单页存更多数据，树更矮。

### 5.5 索引条件下推（ICP）

**Index Condition Pushdown**，MySQL 5.6+ 引入，5.7 默认开启。

```sql
SELECT * FROM t_multiple_index WHERE a = 13 AND b >= 15 AND c = 5 AND d = 'pdf';
```

优化器使用 `idx_abc(a,b,c)`，但 `c` 不满足最左前缀的「精确匹配」——ICP 允许在**存储引擎层**对索引中包含的列（如 c）提前过滤，减少回表。

```sql
-- 查看 ICP 开关
SHOW VARIABLES LIKE 'optimizer_switch';

SET optimizer_switch = 'index_condition_pushdown=off';  -- 关闭
SET optimizer_switch = 'index_condition_pushdown=on';   -- 开启
```

![ICP 原理示意](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-012-icp-diagram.png)

**不使用 ICP**（`Extra: Using where`）：

![不使用 ICP 的回表过程](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-013-icp-without.png)

每条候选记录先回表，再由 Server 层用 `c=5 AND d='pdf'` 过滤 → 回表 3 次，过滤 3 次。

**使用 ICP**（`Extra: Using index condition`）：

在存储引擎层先用 `c=5` 过滤，不满足则不回表 → 回表 2 次，Server 层只需过滤 `d='pdf'`。

| 对比 | 索引条件比较 | 非索引条件比较 |
|------|-------------|---------------|
| 不用 ICP | 存储引擎 | Server 层 |
| 用 ICP | 存储引擎（含部分索引列） | Server 层 |

ICP 仅用于 InnoDB / MyISAM 的**辅助索引**，聚簇索引不需要。

---

## 六、索引创建原则

### 6.1 哪些情况需要创建索引

1. 频繁出现在 `WHERE`、`ORDER BY`、`GROUP BY` 的列。
2. 频繁 `SELECT` 的列，考虑组合索引实现覆盖索引。
3. 多表 `JOIN` 时，`ON` 两侧字段都应建索引。

### 6.2 索引优化建议

| 建议 | 原因 |
|------|------|
| 小表可不建索引 | 全表扫描成本可能更低 |
| 控制索引数量 | 每个索引都是一棵 B+ 树，写操作需全部维护 |
| 避免频繁更新列建索引 | 引发页分裂/合并，写放大 |
| 区分度低的列慎建单列索引 | 如性别、状态，扫描行数可能接近全表 |
| 主键用自增整型 | 避免 UUID 等无序值导致页分裂 |
| 主键/辅助索引键尽量短 | 页容纳更多索引项，减少 IO |
| 优先组合索引 | 一棵索引等价多棵单列索引，省空间、利覆盖 |

组合索引列顺序：**使用频繁 + 区分度高**的列靠前。

---

## 附：索引失效分析

### 案例环境

```sql
CREATE TABLE staffs (
  id      INT NOT NULL AUTO_INCREMENT,
  name    VARCHAR(24) NOT NULL,
  age     INT NOT NULL,
  pos     VARCHAR(20) NOT NULL,
  add_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_staffs_nameAgePos (name, age, pos)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

![staffs 表索引结构](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-014-show-index-staffs.png)

组合索引 `idx_staffs_nameAgePos(name, age, pos)` 按 name → age → pos 排序。

### 1）全值匹配我最爱

```sql
EXPLAIN SELECT * FROM staffs WHERE name = 'july' AND age = 23 AND pos = 'dev';
-- type: ref，key: idx_staffs_nameAgePos，rows 很少
```

三列全部精确匹配，组合索引利用率最高。

### 2）最佳左前缀法则

查询必须从索引**最左列**开始，不能跳过中间列。

```sql
-- 能用索引
EXPLAIN SELECT * FROM staffs WHERE name = 'july' AND age = 23;
EXPLAIN SELECT * FROM staffs WHERE name = 'july';

-- 不能用索引（带头大哥死了）
EXPLAIN SELECT * FROM staffs WHERE age = 23 AND pos = 'dev';

-- 部分失效（中间兄弟断了，pos 不能用索引）
EXPLAIN SELECT * FROM staffs WHERE name = 'july' AND pos = 'dev';
```

![左前缀法则违反示例](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-015-left-prefix-violation.png)

### 3）不要在索引列上做计算

对索引列做函数、运算、隐式类型转换会导致索引失效：

```sql
-- 索引失效：对 name 做了函数
EXPLAIN SELECT * FROM staffs WHERE SUBSTR(name, 1, 3) = 'jul';

-- 索引失效：隐式类型转换（字符串列与数字比较）
EXPLAIN SELECT * FROM staffs WHERE name = 123;
```

### 4）范围条件右边的列失效

```sql
EXPLAIN SELECT * FROM staffs WHERE name = 'july' AND age > 22 AND pos = 'dev';
-- name 走索引，age 范围扫描，pos 无法利用索引定位
```

![范围条件右侧列失效](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-016-range-right-invalid.png)

### 5）尽量使用覆盖索引

```sql
EXPLAIN SELECT name, age, pos FROM staffs WHERE name = 'july' AND age = 23;
-- Extra: Using index（无需回表）
```

避免 `SELECT *`，只查索引中已有的列。

### 6）索引字段上使用不等于

```sql
EXPLAIN SELECT * FROM staffs WHERE name != 'july';
-- 通常 type: ALL 或 range，优化器倾向全表扫描
```

`!=`、`<>` 在大多数情况下难以高效利用 B+ 树有序性。少数场景可能走 `index` 全索引扫描，以 `EXPLAIN` 为准。

### 7）索引字段上判断 NULL

```sql
EXPLAIN SELECT * FROM staffs WHERE name IS NULL;
EXPLAIN SELECT * FROM staffs WHERE name IS NOT NULL;
```

在旧版本 MySQL 中常导致全表扫描。MySQL 8.0+ 对 `IS NULL` 有一定优化，但仍建议用 `EXPLAIN` 验证，不要死记「一定失效」。

### 8）LIKE 以通配符开头

```sql
-- 索引失效
EXPLAIN SELECT * FROM staffs WHERE name LIKE '%july%';
EXPLAIN SELECT * FROM staffs WHERE name LIKE '%july';

-- 索引可用（前缀匹配）
EXPLAIN SELECT * FROM staffs WHERE name LIKE 'july%';
```

![LIKE 通配符与索引](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-017-like-wildcard.png)

`LIKE 'july%'` 相当于范围查询，不会导致右侧列失效（与 `BETWEEN` 不同）。

**`LIKE '%abc%'` 优化思路**：覆盖索引（只查索引列）、全文索引、Elasticsearch。

### 9）字符串不加单引号

```sql
-- 索引失效：name 是字符串，123 触发隐式转换
EXPLAIN SELECT * FROM staffs WHERE name = 123;

-- 正确写法
EXPLAIN SELECT * FROM staffs WHERE name = '123';
```

### 10）索引字段使用 OR

```sql
-- 通常索引失效，走全表扫描
EXPLAIN SELECT * FROM staffs WHERE name = 'july' OR name = 'Tom';
```

![OR 导致索引失效](https://qijian-1301807797.cos.ap-guangzhou.myqcloud.com/markdown/mysql-index/mysql-index-018-or-index-fail.png)

**改写方案**：

```sql
-- 改用 UNION（各自走索引再合并）
SELECT * FROM staffs WHERE name = 'july'
UNION
SELECT * FROM staffs WHERE name = 'Tom';

-- 或确保 OR 两侧都有独立索引，优化器可能用 index_merge
```

### 总结

| 口诀 | 核心含义 |
|------|----------|
| 全值匹配我最爱 | 组合索引列全部精确匹配最优 |
| 最左前缀要遵守 | 从索引第一列开始匹配 |
| 带头大哥不能死 | 缺少最左列则索引不可用 |
| 中间兄弟不能断 | 跳过中间列，右侧列失效 |
| 索引列上不计算 | 函数/运算/隐式转换破坏索引 |
| 范围之后全失效 | 范围条件右侧列无法用于定位 |
| Like 百分写最右 | 前缀通配可用，`%` 开头不行 |
| 覆盖索引不写星 | 避免 `SELECT *`，只查索引列 |
| 不等空值还有 OR | 慎用 `!=`、`= NULL`、`OR` |

> 索引失效规则不是绝对真理，MySQL 优化器会基于成本选择执行计划。**养成习惯：每条可疑 SQL 都跑 `EXPLAIN`**，以 `type`、`key`、`rows`、`Extra` 为准。
