---
title: 浅谈 ConcurrentHashMap
date: 2020/5/30 23:41:0
tags: [Java,Java 集合框架]
categories: [Java 集合框架]
---

我们知道，HashMap 是非线程安全的，在多线程环境下我们可能需要使用线程安全的 Map 容器，可选的有 HashTable 和 ConcurrentHashMap。HashTable 将所有可能出现线程安全问题的方法都使用 synchronized 来修饰，这种做法虽然简单粗暴，但是由于锁的粒度较大（所有访问 HashTable 的线程都在竞争同一把锁），导致它的并发性能较差。而 ConcurrentHashMap 则不同，在 JDK 7 中，它将锁细化为多把锁，每一把锁只用于锁定容器中的一部分数据，那么当多线程访问容器中不同数据段的数据时，线程之间是不存在锁竞争的，这就是我们常说的锁分段技术。但是到了 JDK 8，ConcurrentHashMap 又出现了很大的变化，最大的变化就是不再使用锁分段技术，转而使用 CAS 和 synchronized，这在文章中会具体介绍。

<!--more-->

# JDK 7
在 JDK 7 中，ConcurrentHashMap 主要是由 Segment 数组和 HashEntry 数组构成。Segment 继承自 ReentrantLock，因此它扮演的是独占锁的角色。HashEntry 类似于 HashMap 中的 `HashMap$Entry<K,V>`，用于存储键值对数据。一个 ConcurrentHashMap 包含一个 Segment 数组，这个数组可以理解成类似 HashMap 中的 table 数组，同时一个 Segment 又包含一个 HashEntry 数组，每个 HashEntry 不仅存储键值对数据，同时还维护着一个指向下一节点的引用，从而形成一个链表。Segment 数组与 HashMap 中的 table 数组的唯一区别就是它们并不存储数据，每个 Segment 都守护着各自的 HashEntry 数组元素，当需要对数组元素进行修改时，必须首先获得与之对应的 Segment 锁。

## 初始化
通过设置初始化容量 initialCapacity、负载因子 loadFactor、预估的并发更新线程数 concurrencyLevel 等几个参数来初始化 Segment 数组、段偏移量 segmentShift、段掩码 segmentMask 和每个 Segment 中的 HashEntry 数组。

```java
public ConcurrentHashMap(int initialCapacity,
                          float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                          (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

Segment 数组的长度 ssize 是通过 concurrencyLevel 计算得出的，为了能够实现按位与运算来定位 Segment 数组的索引，必须保证数组的长度是 2 的 N 次方，因此不管传入的 concurrencyLevel 是多少，总能够计算出一个大于或者等于 concurrencyLevel 的最小的 2 的 N 次方的值来作为 Segment 数组的长度。如果 concurrencyLevel 为 14、15 或者 16，那么计算出的数组长度就是 16。需要注意的是，Segment 数组在初始化之后就是固定的，无法扩容，同时 concurrencyLevel 的最大值是 65535，对应的二进制是 16 位。