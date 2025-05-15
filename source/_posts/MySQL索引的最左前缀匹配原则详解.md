---
title: MySQL索引的最左前缀匹配原则详解
date: 2025-04-22 13:50:59
top_group_index: 1
categories: 后端
tags: [mysql, 索引, 最左前缀匹配原则]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/08E1A02625DBA8EB827B5BC84AFDCB83.jpg
---
# MySQL索引的最左前缀匹配原则详解

## 一、最左前缀匹配原则的定义

最左前缀匹配原则(Leftmost Prefix Principle)是MySQL使用联合索引(复合索引)时的基本规则，指的是**查询条件必须从联合索引的最左列开始，并且不能跳过中间的列**，才能充分利用索引。

## 二、核心要点

1. **从最左列开始**：查询条件必须包含联合索引的第一列
2. **连续使用**：可以只使用索引的前几列，但不能跳过中间的列
3. **范围查询后的列失效**：某一列使用范围查询(>、<、between等)后，其右边的列无法使用索引

## 三、具体示例分析

假设有联合索引 `INDEX(a, b, c)`：

### 有效使用索引的情况：

```sql
WHERE a = 1 AND b = 2 AND c = 3  -- 完全使用索引
WHERE a = 1 AND b = 2             -- 使用a,b列
WHERE a = 1                        -- 只使用a列
WHERE a = 1 AND c = 3              -- 只使用a列(c列被跳过)
WHERE a > 1 AND b = 2              -- 只使用a列(范围查询后b失效)
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 无法使用索引的情况：

```sql
WHERE b = 2                        -- 缺少最左列a
WHERE b = 2 AND c = 3              -- 缺少最左列a
WHERE a = 1 AND c = 3 AND b > 2    -- b列范围查询后c失效
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 四、特殊场景说明

1. **LIKE语句**：

   ```sql
   WHERE a LIKE '张%'     -- 可以使用索引(前缀匹配)
   WHERE a LIKE '%张'     -- 不能使用索引
   ```

   ![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

2. **IS NULL/IS NOT NULL**：

   ```sql
   WHERE a IS NULL       -- 可以使用索引
   ```

   ![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

3. **函数和表达式**：

   ```sql
   WHERE UPPER(a) = 'ABC' -- 索引失效(对列使用函数)
   WHERE a + 1 = 2        -- 索引失效(对列使用表达式)
   ```

   ![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 五、实际应用建议

1. **设计索引时**：

   - 将选择性高的列放在左边
   - 常用查询条件列尽量包含在索引最左端

2. **编写SQL时**：

   - 确保查询条件包含索引最左列
   - 避免在索引列上使用函数或计算
   - 范围查询尽量放在最后

3. **排序和分组优化**：

   ```sql
   -- 可以充分利用(a,b,c)索引
   ORDER BY a, b, c
   GROUP BY a, b, c
   
   -- 无法充分利用索引
   ORDER BY b, c
   ```

   ![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 六、原理探究

MySQL的B+树索引按照索引定义的列顺序构建：

1. 先按第一列排序
2. 第一列相同再按第二列排序
3. 以此类推...

因此，跳过前面的列就相当于在一个无序的数据集中查找，无法利用索引的有序性。

## 七、验证方法

使用`EXPLAIN`查看执行计划，关注：

- `type`列为`ref`或`range`表示使用了索引
- `key`列显示实际使用的索引
- `key_len`显示实际使用的索引长度
