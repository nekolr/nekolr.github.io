---
title: 浅谈 ConcurrentHashMap
date: 2020/5/30 23:41:0
tags: [Java,Java 集合框架]
categories: [Java 集合框架]
---

我们知道，HashMap 是非线程安全的，在多线程环境下我们可能需要使用线程安全的 Map 容器，可选的有 HashTable 和 ConcurrentHashMap。HashTable 将所有可能出现线程安全问题的方法都使用 synchronized 来修饰，这种做法虽然简单粗暴，但是由于锁的粒度较大（所有访问 HashTable 的线程都在竞争同一把锁），导致它的并发性能较差。而 ConcurrentHashMap 则不同，在 JDK 7 中，它将锁细化为多把锁，每一把锁只用于锁定容器中的一部分数据，那么当多线程访问容器中不同数据段的数据时，线程之间是不存在锁竞争的，这就是我们常说的锁分段技术。但是到了 JDK 8，ConcurrentHashMap 又出现了很大的变化，最大的变化就是不再使用锁分段技术，转而使用 CAS 和 synchronized，这在文章中会具体介绍。

<!--more-->

# JDK 7
在 JDK 7 中，ConcurrentHashMap 主要是由 Segment 数组和 HashEntry 数组构成。Segment 继承自 ReentrantLock，因此它扮演的是独占锁的角色。HashEntry 类似于 HashMap 中的 `HashMap$Entry<K,V>`，用于存储键值对数据。一个 ConcurrentHashMap 包含一个 Segment 数组，这个数组可以理解成类似 HashMap 中的 table 数组，同时一个 Segment 又包含一个 HashEntry 数组，每个 HashEntry 不仅存储键值对数据，同时还维护着指向下一节点的引用，从而形成一个链表。每个 Segment 都守护着各自的 HashEntry 数组元素，当需要对数组元素进行修改时，必须首先获得与之对应的 Segment 锁。

![ConcurrentHashMap](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202006011913/2020/06/01/oPL.png)

## 初始化
通过设置初始化容量 initialCapacity、负载因子 loadFactor、预估的并发更新线程数 concurrencyLevel 等几个参数来初始化 Segment 数组、段偏移量 segmentShift、段掩码 segmentMask 和每个 Segment 中的 HashEntry 数组。

```java
public ConcurrentHashMap(int initialCapacity,
                          float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
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
    // cap 最小为 2
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // 创建 segments 数组并初始化第一个数组元素
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                          (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

Segment 数组的长度 ssize 是通过 concurrencyLevel 计算得出的，为了能够实现按位与运算来定位 Segment 数组的索引，必须保证数组的长度是 2 的 N 次方，因此不管传入的 concurrencyLevel 是多少，总能够计算出一个大于或者等于 concurrencyLevel 的最小的 2 的 N 次方的值来作为 Segment 数组的长度。如果 concurrencyLevel 为 14、15 或者 16，那么计算出的数组长度就是 16。需要注意的是，Segment 数组在初始化之后就是固定的，无法扩容，同时 concurrencyLevel 的最大值是 65536，对应的二进制是 16 位。

默认情况下 concurrencyLevel 为 16，此时 ssize 的值也是 16，而 sshift 等于 ssize 从 1 向左移位的次数，因此默认情况下 sshift 的值为 4。segmentShift 的作用是定位参与计算索引的 hash 值的位数，同时由于 hash() 方法返回值的类型是 int，这就意味着 hash 值是 32 位的，因此 segmentShift 等于 32 - sshift，也就是说在默认情况下 hash 值在参与计算数组索引时需要右移 28 位，即在计算索引时只使用 hash 值的高 4 位参与运算。

## put
```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    // 计算 segments 数组索引
    int j = (hash >>> segmentShift) & segmentMask;
    // 对应的 Segment 如果没有初始化会进行初始化
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
          (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // 定位 HashEntry 数组的索引
        int index = (tab.length - 1) & hash;
        // 获取对应索引位置的 HashEntry
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            // 当前位置已经被占用
            if (e != null) {
                K k;
                // 根据 key 的情况决定是否覆盖新值
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    // 扩容方法
                    // 只对某个 Segment 进行两倍扩容
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

## get
```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                  (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
              e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

get 操作不需要加锁，原因是用于存储值的 value 是使用 volatile 修饰的，根据 Java 内存模型的 happens-before 原则，对 volatile 字段的写入操作先于读取操作，所以我们总能读到最新的值。

## size
如果要统计整个 ConcurrentHashMap 的大小，就必须统计所有 Segment 里元素的个数后求和。Segment 中的成员变量 count 存储的就是 Segment 中元素的个数，同时它还使用 volatile 进行了修饰，那么我们是不是将所有 Segment 中的 count 相加就可以了呢？答案是不行，虽然我们在相加时可以获取到 count 的最新值，但是可能会出现在相加的过程中已经使用过的 count 值发生变化，那么统计结果就不准确了。因此最安全的做法就是在统计 size 的时候锁住所有可能引起 size 变化的方法，比如 put、remove 等，但是这种做法显然不够理想。现实中在累加 count 的过程中出现使用过的 count 发生变化的几率很小，因此 ConcurrentHashMap 的做法是先尝试 2 次不通过锁住 Segment 的方式来统计各个 Segment 的大小，如果统计过程中容器的 count 的发生了变化（通过检查 modCount 是否发生变化来实现，在 put、remove 和 clean 方法中操作元素前都会将 modCount 加 1），那么再通过加锁的方式来统计所有 Segment 的大小。

# JDK 8
在 JDK 8 中，ConcurrentHashMap 不再使用 Segment（保留了相关的代码以实现序列化时的兼容性），而是采用 CAS + synchronized 来保证并发情况下的线程安全。其中一个原因是分段锁比较浪费内存空间，毕竟比普通的 Map 多了一个 Segment 数组。同时在生产环境中，同一时刻多个线程竞争同一把锁的概率非常小，这也就意味着同一时刻数组的某个位置被多个线程竞争的概率非常小。

## put
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 不允许插入为 null 的键和值，HashMap 允许插入
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 初始化 table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 定位元素的位置，如果该位置上没有元素，则通过 CAS 进行添加
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                          new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 检查到内部是否正在进行扩容
        else if ((fh = f.hash) == MOVED)
            // 协助进行扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 锁住链表或者红黑树的头节点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // 表示这是链表节点
                    if (fh >= 0) {
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 存在该节点，则更新 value
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                  (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 不存在则在链表尾部添加节点
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 是红黑树的节点
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 如果链表节点个数超过阈值 8，则将链表转化成红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 将 size 的数量加 1，并在其中检查是否需要扩容
    addCount(1L, binCount);
    return null;
}
```

可以看到，如果索引位置没有元素，则直接通过 CAS 自旋的方式确保链表的首节点一定能成功添加，而只有在索引位置存在元素时才会进行加锁，锁对象是该位置的链表或红黑树的头节点。当 table 的容量不足时需要对 table 进行扩容，整个扩容分为两步。第一步是创建一个 nextTable，大小为 table 的两倍。第二步就是将 table 中的数据复制到 nextTable 中。在单线程环境下要实现这两个步骤是很容易的，但是在多线程环境下实现起来就比较麻烦，因为扩容也有可能出现并发执行的情况，因此创建和初始化数组的操作必然只能由一个线程来执行，而数据复制则可以支持并发复制，这样性能可以提升很多，但是相对的复杂度也提升了。