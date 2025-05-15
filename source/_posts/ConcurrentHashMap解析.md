---
title: ConcurrentHashMap解析
date: 2025-05-09 12:21:07
top_group_index: 1
categories: java
tags: [java, 集合]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/411C651857A92B91C951C65A93070D03.jpeg
---

## ConcurrentHashMap解析

以下是对 Java 8 及以后版本 `ConcurrentHashMap` 源码的深入解析，涵盖其底层数据结构、并发控制机制、核心操作流程、扩容与迁移、树化/退化策略，以及性能特性。总体来说，`ConcurrentHashMap` 在 JDK 8 中摒弃了原有的 Segment 分段锁，采用了「数组 + 链表/红黑树 + CAS + synchronized」的设计，使得检索无锁、更新高并发，且在扩容时支持多线程协作，以此在确保线程安全的同时，最大化地提升性能和可伸缩性。

### 底层数据结构

`ConcurrentHashMap` 维护一个类型为 `Node<K,V>[] table` 的主数组，数组长度始终为 2 的幂次方，每个桶（bin）最初存储 `Node` 节点（单链表）或 `TreeBin`（红黑树）

- **Node**：包含 `final int hash; final K key; volatile V val; volatile Node<K,V> next;`，用于链表结构的实现
- **TreeBin**：当单链表长度 ≥ 8 且数组容量 ≥ 64 时，链表会转换为红黑树节点（`TreeNode`），以将单桶最坏查找从 O(n) 降至 O(log n)

### 并发控制机制

#### CAS 与 Unsafe

`ConcurrentHashMap` 使用 `sun.misc.Unsafe` 提供的 CAS 原语（`casTabAt`、`U.compareAndSwapObject` 等）来在无锁情况下完成桶头的插入（桶中无数据时）与更新，从而实现高效的并发写

当 CAS 插入失败（即已存在元素，或链表/树操作），会**对该桶首节点** `**f**` **加** `**synchronized(f)**` **锁以串行化**后续对链表或树结构的修改，保证冲突处理的正确性而不会全表加锁

### 核心操作流程

#### 初始化

- 构造 `ConcurrentHashMap` 实例时，`table` 并不立即分配；首个插入或显式调用 `put` 时，才**通过 CAS** 确保单线程完成初始化（容量默认为 16）

#### `get`

- 直接读取 `table`（不加锁）；
- 计算扰动哈希 `h = spread(key.hashCode())` 并定位桶索引 `i = (n – 1) & h`；
- 若 `tab[i] == null`，返回 `null`；若首节点匹配则返回值；否则依次遍历链表或在 `TreeBin` 中调用 `find` 进行对数级查找

#### `put`

1. 计算哈希并尝试在对应桶通过 CAS 插入新 `Node` 作为头节点；
2. 若 CAS 失败且 `tab[i]` 已存在：

- - 对首节点 `f` 加锁，检查键是否已存在，存在则覆盖；
  - 否则将新节点追加到链表尾部，若链表长度达 8 且表容量 ≥ 64，则调用 `treeifyBin` 将桶转为红黑树；

1. 更新计数器 `size`，如超过阈值 `threshold = capacity * loadFactor`，触发扩容

#### `remove` 与 `compute`

- `remove` 同样通过 CAS 快速尝试删除头节点，失败则锁定对应桶进行遍历删除；
- `computeIfAbsent` 等高级方法在锁定桶头后进行 `remappingFunction` 调用，并在必要时插入新节点，递归调用时会触发 OpenJDK 中对递归的防护机制（过于严格时可参考 JDK-8294891 报告）

### 扩容与迁移

扩容由 `transfer` 方法完成，支持多线程协作：

1. **单线程初始化** `**nextTable**`（容量翻倍）并设置 `transferIndex`；
2. **多线程迁移**：各线程通过 CAS 递减 `transferIndex` 分配任务块，遇到 `forwarding` 标记则跳过，遍历旧 `Node` 并按高位拆分到新表的 `i` 或 `i + oldCap` 位置；
3. 迁移完成后，`nextTable` 替代 `table`，更新 `sizeCtl` 为新容量 × 0.75

### 树化与退化

- **树化阈值**：链表长度 ≥ 8 且表容量 ≥ 64 时由 `treeifyBin` 转为红黑树；
- **退化阈值**：在迁移过程中，若树节点数 < 6，则调用 `untreeify` 将 `TreeBin` 转回链表，以减少平衡维护开销

### 性能与特性

- **并发级别**：检索无锁（完全并发），更新高预期并发性（局部 synchronized + CAS）
- **时间复杂度**：平均 O(1)，最坏 O(log n)（树化后）；
- **内存开销**：红黑树节点需额外维护父/左右指针与颜色标志；
- **线程安全**：内部确保了多线程访问安全，无死锁风险，但不支持全表锁定；全表级别的操作需外部同步或使用其他并发集合。

###  CAS  和Synchronized的使用时机

### 1. **CAS（Compare-And-Swap）使用场景** 🔄

#### 适用于**无冲突**或**低冲突**操作，确保乐观并发性。

- **表初始化**：

- - 使用 CAS 保证 table 初始化只由一个线程成功（`initTable()` 方法）。
  - `casTabAt(tab, i, null, newNode)` 用于无锁创建桶。

- **节点插入（无冲突时）**：

- - 当目标桶 `tab[i]` 为空，直接使用 CAS 插入新节点，无需加锁。

- **计数器更新**：

- - 使用 CAS 递增 `baseCount` 计数器（统计元素个数）。
  - 如果 CAS 失败，才会退化到 `fullAddCount()`，分散计数压力。

#### 典型源码片段：

```java
if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null))) {
    break; // 成功插入
}
```

------

### 2. **synchronized 使用场景** 🔒

#### 适用于**存在冲突**或**复杂结构变更**的操作，确保悲观锁定。

- **桶内链表/树操作**：

- - 如果 CAS 插入失败（表示桶非空），需要遍历链表或红黑树；
  - 此时，对桶头节点 `synchronized (f)` 加锁，串行化冲突修改。

- **链表转红黑树**（treeify）：

- - 当桶中链表长度 ≥ 8，需要加锁完成链表到红黑树的结构变换。

- **红黑树操作**：

- - `TreeBin` 内部操作，比如插入、删除、旋转、平衡时，用 `synchronized` 确保树结构正确。

- **复杂删除和 compute 方法**：

- - 如 `remove()`、`computeIfAbsent()`、`compute()`，涉及读取 + 修改；
  - 这些方法会对桶头 `synchronized` 来串行化整个修改逻辑。

#### 典型源码片段：

```java
synchronized (f) {
    Node<K,V>[] tab2;
    // 遍历链表/树进行插入或更新
}
```

------

#### ✅ 总结记忆口诀：

| 场景                     | 用法             |
| ------------------------ | ---------------- |
| **无冲突，简单插入**     | **CAS**          |
| **冲突、遍历、结构变更** | **synchronized** |
| **链表 → 红黑树转化**    | synchronized     |
| **红黑树插入/删除**      | synchronized     |
| **size 计数更新**        | CAS + 分段计数   |

------

#### 🚩设计哲学：

- **优先使用 CAS（乐观并发）** → 快速失败重试，效率高；
- **结构性修改退化为 synchronized** → 保证链表/树一致性，不追求无锁，但控制在**桶级别**，避免全表锁。
