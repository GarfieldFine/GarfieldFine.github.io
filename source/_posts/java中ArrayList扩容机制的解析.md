---
title: java中ArrayList扩容机制的解析.md
date: 2025-05-07 21:10:24
top_group_index: 1
categories: java
tags: [java, 集合]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/54A4D606B65EB00661ED6EFC7945705D.jpeg
---
# java中ArrayList扩容机制的解析

本文将系统地介绍 Java 中 `ArrayList` 的扩容机制，包括其初始容量的设置、触发扩容的时机、容量增长算法、扩容的详细流程以及性能优化建议，帮助读者从源码层面深入理解这一关键特性，并在实际开发中合理预分配容量以提升性能。

## 一、ArrayList 简介

`ArrayList` 底层由一个可变长度的数组 `elementData` 支撑，可在运行时动态调整大小，属于顺序表的具体实现，具有随机访问速度快、增删效率相对较低、非线程安全的特点 

## 二、初始化与构造

### 1. 无参构造器

- 调用 `new ArrayList<>()` 时，`elementData` 被赋值为一个空数组常量 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，此时容量为 0
- 当第一次添加元素时，`ensureCapacityInternal(1)` 将最小容量 `minCapacity` 与默认容量 `DEFAULT_CAPACITY = 10` 比较，最终分配容量为 10

### 2. 指定初始容量构造器

- 使用 `new ArrayList<>(initialCapacity)` 会直接分配长度为 `initialCapacity` 的底层数组，跳过默认容量提升逻辑，可避免第一次添加元素时的数组复制

## 三、扩容触发时机

- 每次调用 `add(E e)` 时，都会执行 `ensureExplicitCapacity(size + 1)`，若 `size + 1` 超出当前数组长度则触发扩容
- `ensureCapacityInternal` 会仅在 `minCapacity - oldCapacity > 0` 时才进行扩容，以避免多余的复制操作 

## 四、扩容算法

- 自 JDK 8 起，容量增长公式为：

  ```
  java
  
  
  复制编辑
  int newCapacity = oldCapacity + (oldCapacity >> 1);
  ```

  即在旧容量基础上增加约 50%。

- 在 JDK 6 及更早版本，采用 `(oldCapacity * 3) / 2 + 1` 的方式，效果相似。

- 若计算出的 `newCapacity` 小于 `minCapacity`，则直接使用 `minCapacity`；若超过 `Integer.MAX_VALUE`，则调用 `hugeCapacity` 调整至可用最大值或抛出 `OutOfMemoryError`，以防止整数溢出。

## 五、扩容流程详解

1. **计算最小需求**

   - `minCapacity = size + 1`。

2. **空数组检测**

   - 若底层数组为 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，则将 `minCapacity` 与 `DEFAULT_CAPACITY`（10）比较，取较大者。

3. **调用 `grow(minCapacity)`**

   - 读取 `oldCapacity = elementData.length`。
   - 按扩容算法计算新容量 `newCapacity = oldCapacity + (oldCapacity >> 1)`。
   - 若 `newCapacity < minCapacity`，则 `newCapacity = minCapacity`。
   - 若 `newCapacity` 超出最大阈值，则经 `hugeCapacity(minCapacity)` 处理。

4. **数组复制**

   - 使用 `Arrays.copyOf(elementData, newCapacity)` 创建并替换底层数组，实现元素迁移
   - **迭代器快速失败支持**

   - 扩容时会自增 `modCount`，保证在多线程环境下对并发修改的检测与抛出 `ConcurrentModificationException`。

## 六、性能优化建议

- **预分配容量**：若能预计元素数量，建议使用 `new ArrayList<>(expectedSize)` 或提前调用 `ensureCapacity(expectedSize)`，可显著减少扩容次数及数组复制开销 。
- **扩容策略权衡**：1.5 倍的增长因子在性能（降低复制频次）与内存利用率（避免过度浪费）之间取得平衡，是常见动态数组设计策略。
- **批量操作**：对于大量元素的插入，可先将所有元素聚合至临时数组或使用 `addAll(Collection)`，以减少多次单个元素添加带来的重复扩容开销.。
