---
title: MySQL 五种存储引擎详解及对比
date: 2025-04-21 09:26:59
top_group_index: 1
categories: 后端
tags: [mysql, 存储引擎]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/D7AC543AC33C11790B7A452D4E3551AA.jpeg
---
# MySQL 五种存储引擎详解及对比

## 1. InnoDB

**特点**：

- MySQL 5.5+ 后的默认存储引擎
- 支持**事务处理**（ACID 兼容）
- 实现**行级锁定**（并发性能好）
- 支持**外键约束**
- 提供**崩溃恢复**能力
- 使用**MVCC**（多版本并发控制）
- 数据存储在**表空间**中（可配置为每表单独文件）
- 支持**热备份**

**适用场景**：

- 需要事务支持的应用程序
- 高并发读写操作
- 需要外键约束的应用
- 大多数 OLTP（在线事务处理）系统

## 2. MyISAM

**特点**：

- MySQL 5.5 之前的默认引擎
- **不支持事务**
- 使用**表级锁定**（并发性能较差）
- 支持**全文索引**（FULLTEXT）
- 较高的**读取速度**
- 存储由**.MYD**（数据）和**.MYI**（索引）文件组成
- 支持**压缩表**（只读）
- 支持**空间数据类型**

**适用场景**：

- 读密集型应用
- 不需要事务的简单应用
- 数据仓库或报表系统
- 需要全文索引的应用（MySQL 5.6 前）

## 3. MEMORY (HEAP)

**特点**：

- 所有数据存储在**内存**中
- **极快的访问速度**
- 使用**表级锁定**
- 不支持**BLOB/TEXT**类型
- 服务器重启后**数据丢失**
- 默认使用**哈希索引**（也支持B树索引）
- 表大小受**max_heap_table_size**参数限制

**适用场景**：

- 临时数据存储
- 缓存中间结果
- 需要极快访问速度的查找表
- 会话管理等临时数据

## 4. NDB (Cluster)

**特点**：

- MySQL **集群**存储引擎
- **高可用性**设计（自动分片和复制）
- 数据存储在**内存**中（可配置为磁盘存储）
- 支持**事务**
- 支持**行级锁定**
- 需要**NDB Cluster**环境
- **JOIN操作性能较差**
- 适合**分布式计算**环境

**适用场景**：

- 需要99.999%可用性的应用
- 电信、实时计费系统
- 需要线性扩展的高负载应用
- 分布式数据库环境

## 5. ARCHIVE

**特点**：

- 专为**高压缩比**设计（比MyISAM小75%）
- 只支持**INSERT**和**SELECT**操作
- **不支持索引**（主键除外）
- **不支持更新/删除**
- 使用**行级锁定**
- 数据压缩后存储，读取时解压
- 适合**只追加**的数据

**适用场景**：

- 日志和审计数据
- 历史归档数据
- 很少访问的大量数据存储
- 数据仓库的底层存储

## 五种引擎对比表

| 特性         | InnoDB  | MyISAM       | MEMORY      | NDB       | ARCHIVE  |
| ------------ | ------- | ------------ | ----------- | --------- | -------- |
| **事务支持** | ✅       | ❌            | ❌           | ✅         | ❌        |
| **锁粒度**   | 行锁    | 表锁         | 表锁        | 行锁      | 行锁     |
| **外键**     | ✅       | ❌            | ❌           | ❌         | ❌        |
| **全文索引** | ✅(5.6+) | ✅            | ❌           | ❌         | ❌        |
| **存储位置** | 磁盘    | 磁盘         | 内存        | 内存/磁盘 | 磁盘     |
| **崩溃恢复** | ✅       | ❌            | ❌           | ✅         | ❌        |
| **压缩**     | 表压缩  | 压缩表(只读) | ❌           | ❌         | 极高压缩 |
| **并发性能** | 高      | 低           | 中          | 极高      | 低       |
| **持久性**   | 持久    | 持久         | 非持久      | 可配置    | 持久     |
| **典型用途** | OLTP    | 读密集型     | 缓存/临时表 | 集群      | 归档数据 |

## 选择建议

1. **常规Web应用**：InnoDB（默认选择）
2. **只读或读多写少**：MyISAM（考虑迁移到InnoDB+从库）
3. **临时数据处理**：MEMORY
4. **电信级高可用**：NDB Cluster
5. **日志归档**：ARCHIVE
