---
title: LinkedHashMap解析
date: 2025-05-09 12:21:07
top_group_index: 1
categories: java
tags: [java, 集合]
cover: https://background-img-1.oss-cn-beijing.aliyuncs.com/F71807886EEF79C6C1B1E8A4DEC41ABF.jpg
---
# LinkedHashMap解析

LinkedHashMap 是 Java 集合框架中在保留 HashMap 高效存取性能的基础上，通过额外维护一条双向链表来保证元素的迭代顺序（插入顺序或访问顺序）的 Map 实现。其核心思路是在每个桶（bucket）节点上都增加 `before` 和 `after` 指针，串联成一个环形双向链表，并由 `head`、`tail` 引用维护链表首尾。当执行 `put`、`get`（若开启访问顺序）、`remove` 等操作时，除了调用 HashMap 的基本操作，还会对链表做相应调整，以保证有序迭代。以下从源码结构、主要方法、链表维护、扩容与迭代等方面做深入剖析。

## 1. 源码结构概览

### 1.1 继承关系与节点设计

- `LinkedHashMap<K,V>` 直接继承自 `HashMap<K,V>`，复用其拉链表/红黑树冲突解决机制，同时为每个节点增加双向链表功能 

- 内部静态类 `LinkedHashMap.Entry<K,V> extends HashMap.Node<K,V>`，在父类 `Node` 的基础上新增两个指针：

  ```java
  Entry<K,V> before; // 前驱
  Entry<K,V> after;  // 后继
  ```

  使得所有节点在 hash 桶结构之外，还串联成一条双向链表 

### 1.2 关键字段

- `transient Entry<K,V> head`：链表的“最旧”节点（首节点）。
- `transient Entry<K,V> tail`：链表的“最新”节点（尾节点）。 
- `final boolean accessOrder`：决定迭代顺序：
  - `false`：保持插入顺序
  - `true`：保持访问顺序（每次 `get`/`put` 或 `putIfAbsent` 会将节点移到链表尾部）

## 2. 插入操作（put）源码分析

### 2.1 调用 HashMap 的 putNode

- `LinkedHashMap` 的 `put(K key, V value)` 方法最终会调用 `HashMap` 的 `putVal`，它在完成哈希桶插入或更新后，返回插入或更新的 `Node`。

### 2.2 链表尾部插入

- 在 `HashMap.putVal` 执行完毕后，`LinkedHashMap` 会重写 `afterNodeInsertion(boolean evict)`：

  ```java
  void afterNodeInsertion(boolean evict) {
      // 新节点 always 添加到 tail 之后
      Entry<K,V> p = (Entry<K,V>) e;
      LinkedHashMap.Entry<K,V> last = tail;
      tail = p;
      if (last == null)
          head = p;
      else {
          p.before = last;
          last.after = p;
      }
      if (evict && removeEldestEntry(head))
          removeNode(head.hash, head.key, null, false, true);
  }
  ```

- 其中 `evict` 用于支持根据策略（如 LRU）剔除最旧节点。该逻辑在 `removeEldestEntry` 返回 `true` 时触发 

## 3. 访问操作（get）及访问顺序维护

### 3.1 默认不改变链表结构

- 若 `accessOrder == false`，`get` 操作仅返回值，不调整链表。

### 3.2 访问顺序下的节点后移

- 若 `accessOrder == true`，在 `HashMap.getNode` 调用后，`LinkedHashMap` 重写了 `afterNodeAccess(Node<K,V> e)`：

  ```java
  void afterNodeAccess(Node<K,V> e) {
      LinkedHashMap.Entry<K,V> last;
      if (accessOrder && (last = tail) != e) {
          LinkedHashMap.Entry<K,V> p = (Entry<K,V>) e;
          // 断链
          unlink(p);
          // 重新接到尾部
          p.before = last;
          p.after = null;
          last.after = p;
          tail = p;
      }
  }
  ```

- 其中 `unlink(p)` 会调整 `before.after`、`after.before` 等指针，确保链表完整性 

## 4. 删除操作与链表维护

### 4.1 普通删除

- `remove(Object key)` 会在 `HashMap` 中删除节点，随后调用 `afterNodeRemoval(Node<K,V> e)`：

  ```java
  void afterNodeRemoval(Node<K,V> e) {
      Entry<K,V> p = (Entry<K,V>) e, b = p.before, a = p.after;
      p.before = p.after = null;
      if (b == null)
          head = a;
      else
          b.after = a;
      if (a == null)
          tail = b;
      else
          a.before = b;
  }
  ```

- 这样可在 O(1) 时间内从双向链表中摘除任意节点 

### 4.2 批量清空

- `clear()` 调用 `HashMap.clear()` 清桶后，同时将 `head = tail = null`，断开所有链表引用。

## 5. 扩容（resize）与迭代顺序

- `HashMap` 的扩容只会重新分配并重链桶内链表/树结构，不会改变 `LinkedHashMap` 维护的双向链表；即原有的 `before`/`after` 关系保持不变 
- 因此，无论何时遍历 `LinkedHashMap`，都能根据链表次序输出所有键值对。

## 6. 小结

1. **结构**：在 `HashMap` 的每个节点基础上，通过 `Entry.before`/`Entry.after` 串成环形双向链表，并用 `head`、`tail` 跟踪首尾 
2. **顺序维护**：
   - **插入顺序**（默认）：新节点追加到尾部。
   - **访问顺序**（`accessOrder=true`）：每次访问将对应节点移动到尾部。
3. **性能**：
   - 与 `HashMap` 相比，插入、删除及访问（若维护顺序）多了 O(1) 的链表操作开销。
   - 保留了哈希表的 O(1) 平均存取性能，并在需要有序迭代时表现优异。

通过以上分析，可以清晰地理解 `LinkedHashMap` 如何在 `HashMap` 基础上，以最小改动代价引入双向链表，进而同时满足快速存取和有序迭代两大需求。
