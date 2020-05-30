---
title: 深入 HashMap
date: 2018/10/17 09:56:0
tags: [Java,Java 集合框架]
categories: [Java 集合框架]
---

在 JDK 1.8 之前，HashMap 底层使用数组 + 链表的方式实现，在 JDK 1.8 中引入了红黑树来优化链表过长的问题，底层结构就变成了数组 + 链表 + 红黑树。  

<!--more-->  

![HashMap](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/10/23/NWp.png)

# 常量

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
```
默认的容量为 16，必须是 2 的幂（与 HashMap 的哈希算法有关）。  

```java
static final int MAXIMUM_CAPACITY = 1 << 30;
```
容量必须是 2 的幂，同时上限为 2^30。  

```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
默认的负载因子为 0.75。  

```java
static final int TREEIFY_THRESHOLD = 8;
```
默认桶内数据结构由链表转换成树的阈值为 8（即桶内 bin 个数达到 8 个）。  

```java
static final int UNTREEIFY_THRESHOLD = 6;
```
默认桶内数据结构由树转换成链表的阈值为 6。  

```java
static final int MIN_TREEIFY_CAPACITY = 64;
```
在转换成树之前，还需要判断键值对的数量，只有数量大于 64 才会转换，数量小于 64 会执行 resize() 进行扩容。这是为了避免在哈希表建立初期，多个键值对恰好被放入了同一个链表中而造成不必要的转化（因为初期完全可以靠扩容解决）。  

# 属性

```java
transient Node<K,V>[] table;
```
在 HashMap 中维护着一个名称为 table 的数组，元素类型为 HashMap$Node<K,V>，Node<K,V> 是 HashMap 的静态内部类，实现了 Map<K,V> 接口的内部接口 Map$Entry<K,V>（在以前的 JDK 版本中，table 数组的元素类型为 HashMap$Entry<K,V>，同样实现了 Map$Entry<K,V> 接口）。  

这个数组中的每个元素的位置我们称为“桶（bucket）”，它储存链表的第一个元素或者是树的根节点。。当放入元素的 key 的 hash 值与数组中某个元素的相同，并且 key 值 equals 为 false 时，将该元素挂在这个位置的元素后面，形成类似链表的结构；如果 key 值 equals 为 true，则直接更新 value。  

```java
transient int size;
```
这个 size 表示的是 HashMap 中键值对的个数（包含桶结构上的节点）。  

```java
final float loadFactor;
```
负载因子是哈希表在其容量自动增长之前可以达到多满的一个尺度，它衡量的是一个散列表的空间使用程度，负载因子越大（最大 1）表示散列表的装填程度越高。  

对于使用链表的散列表来说，负载因子越大，空间利用程度越高，但是相对的冲突的可能性增大，查找的效率会降低（最坏的情况是全部冲突，退化成线性查找，时间复杂度为 O(n)）；负载因子越小，空间利用程度越低，但是冲突的可能性减小，查找的效率会提高（最好的情况是没有冲突，这样时间复杂度为 O(1)）。  

```java
int threshold;
```
要调整容量大小的下一个阈值（通过 capacity * loadFactor 可以计算得出）。  

# 构造方法

HashMap 共有四个构造函数。  

- 第一个

```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

使用无参构造函数，会初始化负载因子为默认的 0.75。在初次放入元素时，会调用 resize() 方法，初始化容量为默认的 16，阈值为 16 * 0.75 = 12，然后创建一个长度为 16 的 HashMap$Node<K,V> 数组赋给 table。  

- 第二个  

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

这里的 `threshold` 并不是使用 `capacity * loadFactor` 计算得到的，而是通过 `tableSizeFor()` 方法得到的。   

```java
/**
* Returns a power of two size for the given target capacity.
*/
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

通过代入值，比如代入 2^29 + 1，计算得出 n + 1 的值为 2^30。所以这个方法是找到大于或者等于 capacity 的最小 2 的幂值。  

- 第三个  

```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

使用默认的负载因子大小 0.75，指定初始容量，当然这个容量会在第一次放入元素时，通过 resize() 方法赋成在构造方法中计算出的 `threshold` 的值。即不管指定的容量是多少，都会被重新计算并赋值为初始容量的最小 2 的幂值。  

- 第四个  

```java
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

此构造方法使用默认的负载因子，将通过一个指定的 Map 构造出一个新的 HashMap。  

# 计算元素位置
我们在 HashMap 中要找到某个元素，需要先找到数组的下标，而 `key.hashCode()` 得到的是 JDK 提供的键的 hash 值，是一个 int 类型的散列值，考虑到 int 值的范围是从 - 2^31 到 2^31 - 1，这个值肯定不能直接作为数组下标使用。  

我们期望这个 HashMap 中的元素分布尽量均匀，最好是每个位置只有一个元素，这样我们通过算法得到的索引位置的元素一定就是我们要找的元素，而不需要再去查找链表或者树。  

我们首先想到的就是将 hash 值对数组长度取模，这样元素的分布相对来说比较均匀，但是取模运算的消耗较大，所以 JDK 中采用如下方式：  

```java
static int indexFor(int h, int length) {
    return h & (length - 1);
}
```

上面的源码是 JDK 1.6 中的，但是在 JDK 1.8 中还是采用的这种方式。因为 HashMap 保证数组的长度一定是 2 的幂，这种情况下数组的长度减一正好相当于一个**低位掩码**，与操作的结果就是散列值的高位全部归零，只保留低位值作为数组的下标。比如当数组长度为 16 时，和某个散列值进行与运算：  

```
    11010101 11010011 01011010 00101101
&   00000000 00000000 00000000 00001111
---------------------------------------
    00000000 00000000 00000000 00001101
```

> 其实当数组长度为 2 的幂时，`h % length` 与 `h & (length - 1)` 等价。  

但是如果我们只使用到了散列值的最后几位的话，即使散列值再松散，碰撞也会很严重，这个时候就需要使用**扰动函数**了。JDK 1.8 中的扰动函数如下：  

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

比如下面这个例子：  

```
h               11010101 11010011 01011010 00101101
h >>> 16        00000000 00000000 11010101 11010011
h ^ (h >>> 16)  11000101 11010011 10001111 11111110
```

由于 int 类型长度为 32 位，所以右移 16 位，正好是 32 位的一半，高位与低位进行异或运算，目的是**为了混合原始哈希值的高位和低位，以此来加大低位的随机性**。混合后的低位混合了部分高位的特征，这样高位的信息也就被变相的保留了下来。  

同时，如果重写了 key 的 hashCode() 方法，可能会因为写法不合理造成 hashCode 分布不均匀导致哈希冲突比较严重，通过移位异或运算，可以让 hash 值变得更加复杂，进而影响 hash 值的分布性。  

# 获取元素
```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // (n - 1) & hash 计算索引位置
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 如果 hash 值相同，并且两个键的内存地址相同或者是两个键 equals 为 true，
        // 则表示找到了该 key
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 否则，查找它是否有下一个节点
        if ((e = first.next) != null) {
            // 如果是树节点，则使用获取树节点的方法
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 遍历链表，直到找到该 key
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

# 插入元素
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果数组没有初始化，则通过 resize() 方法进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 计算键值对将要放入的位置，如果该位置没有其他节点则通过
    // newNode() 方法创建节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 如果该位置上有其他节点
    else {
        Node<K,V> e; K k;
        // 如果 hash 值相同，并且是同一个键对象或者两个键 equals 为 true
        // 表示当前 HashMap 已经存在要插入的键值对
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果该位置上是树形节点，则调用树形节点的插入方法
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 使用循环统计桶结构上节点的数量
            for (int binCount = 0; ; ++binCount) {
                // 如果该位置的下一个节点为空，则将要插入的元素放在该节点的后面
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 当桶结构上的节点数量超过链表转树的阈值时，进行转换
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果下一个节点不为空，hash 值相同并且是同一个键对象或者两个键
                // equals 为 true，表示当前的 HashMap 中存在要插入的键值对
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // e 不为空表示 HashMap 中已经存在要插入的键值对
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // onlyIfAbsent 表示仅在 oldValue 为空的时候才更新键值对的值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 键值对的数量超过阈值，则进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

插入元素主要逻辑包括以下几点：  

1. 当数组为空时，使用 resize() 方法扩容来初始化数组。  
2. 查找需要插入的键值对是否已经存在，如果存在，则需要根据 onlyIfAbsent 来决定是否要覆盖值。  
3. 如果不存在，并且不存在冲突，则直接放入元素；如果存在冲突则将键值对链接到链表上，同时当链表长度超过阈值时会转换成红黑树。  
4. 判断键值对的数量，大于阈值会进行扩容。  

# 删除元素
```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 数组不为空并且该位置上有元素
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // key 匹配
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // 获取树节点
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 遍历链表
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 找到节点
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            // 如果是树节点，则调用删除树节点的方法
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 数组元素置为空
            else if (node == p)
                tab[index] = node.next;
            // 链表删除节点，直接指向下一个节点
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

# 扩容
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    // 旧的容量
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 旧的阈值
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 旧的容量大于等于最大容量时，修改阈值为 Integer 的最大值，
        // 同时不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 旧容量扩容为原来的 2 倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 阈值同样扩容为原来的 2 倍
            newThr = oldThr << 1; // double threshold
    }
    // 将旧的阈值指定为新的容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 调用无参构造时初始化的容量和阈值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 按阈值计算公式计算
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    // 直接创建新容量的数组
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 遍历旧的数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 数组元素不为空，则进行元素 rehash
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 该位置元素没有下一个节点，则直接计算
                // 该元素在新数组中的位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果是树，则需要对树进行拆分
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 遍历链表
                    do {
                        // 下一个节点
                        next = e.next;
                        // 如果 hash & oldCap 为 0
                        if ((e.hash & oldCap) == 0) {
                            // 如果尾节点为空，即链表为空，则
                            // 将节点赋给头节点和尾节点
                            if (loTail == null)
                                loHead = e;
                            // 如果还有 hash & oldCap 为 0 的
                            // 节点，则继续向后链入
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 如果 hash & oldCap 不为 0，则
                        // 类似的将节点链入另一个链表 hiHead
                        // 和 hiTail 中
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        // 将尾部置空
                        loTail.next = null;
                        // 将头部指向新数组的该位置
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 将头部指向新数组的另一个位置
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

扩容主要做了以下工作：  

1. 计算新数组的容量和阈值。  
2. 根据新的容量直接创建一个新的数组。  
3. 将键值对重新映射到新数组中。如果是树节点，则需要拆分红黑树；如果是普通节点，则直接 rehash 进行映射；如果是链表，则不需要重新进行多次的 rehash，而是将链表根据 hash & oldCap 进行分组，并将分组后的两个链表分别链入新的数组中。  

这里需要重点分析的就是链表的分组。如果旧数组上存在链表，一般的做法是遍历链表，然后一个个 rehash 计算它们在新数组的位置，在 JDK 1.7 中确实是这样做的。但是在 JDK 1.8 中则优化了这种做法，采用的就是根据 `hash & oldCap == 0` 来对链表节点进行分组，下面举个例子来说明。  

假如初始容量为 16，数组上存在链表 5 -> 21 -> 37 -> 53。  

```
5       0000 0101
21      0001 0101
37      0010 0101
53      0011 0101
&
16-1    0000 1111
```

可以看到，它们与容量减一进行与运算的结果都是 0101。当容量扩容为 32 时，此时原链表上的节点再进行与运算就会出现不同的索引值。  

```
5       0000 0101
21      0001 0101
37      0010 0101
53      0011 0101
&
32-1    0001 1111
```

但是其实并不需要重新计算每个节点的索引值，只需要判断因为扩容导致左侧多出的那一位的值。如果是 1，则需要将节点链接到 newTab[j + oldCap] 上；如果是 0，则原地不动。这也就是为什么使用 `hash & oldCap == 0` 作为分组判断条件的原因。  

# 序列化
如果仔细阅读源码，你会发现 HashMap 的桶数组 table 被修饰成了 `transient` 的。  

```java
transient Node<K,V>[] table;
```

在 Java 中，被该关键字修饰的变量不会被默认的序列化机制序列化，但是我们肯定需要用到 HashMap 的序列化功能，那该怎么办呢？其实 HashMap 自己实现了自定义的序列化功能，即实现了 `readObject()` 和 `writeObject()` 这两个方法。为什么 HashMap 不使用默认的序列化机制，而是选择自定义实现呢？这其实主要有两方面的原因：  

其中一个原因是 table 在多数情况下是无法被存满的，序列化未使用的部分会造成空间的浪费。另一个原因是同一个键值对在不同的 JVM 下所处的桶位置可能不一样，这可能会造成不同的 JVM 反序列化时出错。因为如果没有重写键的 hashCode() 方法，则会使用 Object 的 hashCode() 方法，它是一个本地方法，不同的 JVM 下可能会有不同的实现，产生的哈希值也就可能不一样。  

# 参考

> [知乎：关于hashMap的一些按位与计算的问题？](https://www.zhihu.com/question/28562088)
