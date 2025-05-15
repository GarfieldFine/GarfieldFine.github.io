---
title: MySQL索引使用一定有效吗？如何排查索引效果？
date: 2025-04-23 16:50:29
top_group_index: 1
categories: 后端
tags: [mysql, 索引]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/020044D11B42EC6DF62DE855E93FDEB2.jpg
---
# **MySQL索引使用一定有效吗？如何排查索引效果？**

## **1. 索引一定有效吗？**

**不一定！** 即使你创建了索引，MySQL 也可能因为以下原因 **不使用索引** 或 **索引效果不佳**：

- **索引选择错误**：MySQL 优化器可能选择了错误的索引。
- **索引失效场景**：某些 SQL 写法会导致索引失效。
- **数据分布问题**：数据量太少或数据分布不均，导致全表扫描更快。
- **索引设计不合理**：索引列顺序、类型不匹配等。

------

## **2. 索引失效的常见场景**

### **(1) 使用 `!=`、`NOT IN`、`NOT EXISTS`**

```
SELECT * FROM users WHERE age != 20;  -- 可能不走索引
SELECT * FROM users WHERE id NOT IN (1, 2, 3);  -- 可能全表扫描
```

### **(2) 使用 `LIKE` 以通配符开头**

```
SELECT * FROM users WHERE name LIKE '%Alice%';  -- 不走索引
SELECT * FROM users WHERE name LIKE 'Alice%';   -- 可能走索引
```

### **(3) 对索引列使用函数或计算**

```
SELECT * FROM users WHERE YEAR(create_time) = 2023;  -- 不走索引
-- 优化：使用范围查询
SELECT * FROM users WHERE create_time BETWEEN '2023-01-01' AND '2023-12-31';
```

### **(4) 数据类型不匹配（隐式转换）**

```
-- name 是 VARCHAR，但用数字查询
SELECT * FROM users WHERE name = 123;  -- 不走索引（隐式转换成字符串）
```

### **(5) 复合索引未遵循最左前缀原则**

```
-- 假设有联合索引 (name, age)
SELECT * FROM users WHERE age = 20;  -- 不走索引（缺少 name 条件）
```

### **(6) 数据量太少**

- 如果表只有几十行数据，MySQL 可能直接全表扫描，因为索引查找+回表的开销更大。

------

## **3. 如何排查索引效果？**

### **方法 1：使用 `EXPLAIN` 分析执行计划**

```
EXPLAIN SELECT * FROM users WHERE name = 'Alice';
```

重点关注：

- **`type`**：查询类型（`const` > `ref` > `range` > `index` > `ALL`）
- **`key`**：实际使用的索引
- **`rows`**：预估扫描行数
- **`Extra`**：额外信息（`Using index` 表示覆盖索引）

**示例输出：**

| id   | select_type | table | type | key      | rows | Extra |
| :--- | :---------- | :---- | :--- | :------- | :--- | :---- |
| 1    | SIMPLE      | users | ref  | idx_name | 1    | NULL  |

- **`type=ALL`**：全表扫描（索引可能未生效）
- **`key=NULL`**：未使用索引

------

### **方法 2：检查索引使用情况**

```
-- 查看表的索引
SHOW INDEX FROM users;

-- 查看索引使用统计（需开启性能模式）
SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage 
WHERE OBJECT_SCHEMA = 'your_db' AND OBJECT_NAME = 'users';
```

- **`index_name`**：索引名称
- **`count_read`**：索引被读取次数（越高说明索引越有效）

------

### **方法 3：使用 `OPTIMIZER_TRACE` 查看优化器决策**

```
-- 开启优化器跟踪
SET optimizer_trace = 'enabled=on';
SET optimizer_trace_max_mem_size = 1000000;

-- 执行查询
SELECT * FROM users WHERE name = 'Alice';

-- 查看优化器决策
SELECT * FROM information_schema.optimizer_trace;
```

- 可以看到 MySQL 为什么选择（或不选择）某个索引。

------

### **方法 4：强制使用索引（测试对比）**

```
-- 强制使用索引
SELECT * FROM users FORCE INDEX (idx_name) WHERE name = 'Alice';

-- 对比性能
EXPLAIN SELECT * FROM users FORCE INDEX (idx_name) WHERE name = 'Alice';
EXPLAIN SELECT * FROM users IGNORE INDEX (idx_name) WHERE name = 'Alice';
```

- 如果强制索引后查询变快，说明优化器可能选错了索引。

------

## **4. 如何优化索引？**

### **(1) 选择合适的索引列**

- **高选择性列**（如 `user_id` 比 `gender` 更适合索引）。
- **频繁查询的列**（如 `WHERE`、`ORDER BY`、`JOIN` 条件）。

### **(2) 使用覆盖索引**

```
-- 优化前：需要回表
SELECT id, name, age FROM users WHERE name = 'Alice';

-- 优化后：使用 (name, age) 联合索引，避免回表
ALTER TABLE users ADD INDEX idx_name_age (name, age);
```

### **(3) 避免索引冗余**

```
-- 已有 (name, age) 索引，再建 (name) 就是冗余的
ALTER TABLE users DROP INDEX idx_name;
```

### **(4) 定期分析表**

```
-- 更新索引统计信息
ANALYZE TABLE users;

-- 重建索引（修复碎片化）
OPTIMIZE TABLE users;
```

------

## **5. 总结**

| 问题               | 解决方案                                     |
| :----------------- | :------------------------------------------- |
| **索引未生效**     | 检查 `EXPLAIN`，避免索引失效场景             |
| **优化器选错索引** | 使用 `FORCE INDEX` 或 `OPTIMIZER_TRACE` 分析 |
| **索引效果差**     | 优化索引设计，使用覆盖索引                   |
| **数据量太少**     | 可能不需要索引                               |

**📌 建议：**

- 使用 `EXPLAIN` 分析关键查询。
- 避免索引失效写法（如 `LIKE '%xxx%`、函数计算）。
- 定期检查索引使用情况，删除冗余索引。
