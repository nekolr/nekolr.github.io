---
title: 深入 LinkedHashMap
date: 2018/10/23 10:56:0
tags: [Java,Java 集合框架]
categories: [Java 集合框架]
---

LinkedHashMap 继承自 HashMap，在 HashMap 的基础上，通过维护一条双向链表，解决了 HashMap 遍历顺序与插入顺序不一致的问题，除此之外还对访问顺序提供了支持，这在一些场景下是很有用的，比如缓存。  

<!--more-->  

![LinkedHashMap](https://img.nekolr.com/images/2018/10/24/RlO.png)

# 节点
其实 LinkedHashMap 的节点并没有什么特别的地方，因为继承自 HashMap，节点类型继承自 HashMap 也是可以理解的。需要说明的是 HashMap 的树节点类型。  

![节点继承关系](https://img.nekolr.com/images/2018/10/24/WRd.png)

LinkedHashMap 在 HashMap 节点的基础上又增加了两个属性 before 和 after，通过这两个属性就可以维护一条按照插入顺序排列的双向链表。但是 HashMap 的树节点继承自 LinkedHashMap 的节点就有点奇怪了，我们在使用 HashMap 的红黑树时，并不需要树节点具有组成双向链表的能力，这样做肯定会造成空间上的浪费。在 HashMap 的源代码中这样一段注释：  

> Because TreeNodes are about twice the size of regular nodes, we use them only when bins contain enough nodes to warrant use (see TREEIFY_THRESHOLD. And when they become too small (due to removal or resizing) they are converted back to plain bins.  In usages with well-distributed user hashCodes, tree bins are rarely used.  

大致的意思是 TreeNode 对象的大小大约是常规 Node 对象的 2 倍，所以我们仅在桶数组中包含足够多的键值对时才使用（TREEIFY_THRESHOLD 的限制）。当桶中的键值对数量变少时（删除元素或者是扩容），TreeNode 会转换成 Node。用户在使用分布均匀的 hashCode 时，红黑树是很少使用的。  

通过这段注释我们可以了解到，只要 hashCode 的实现不太糟糕，Node 是很少会转换成 TreeNode 的，并且如果 TreeNode 继承自 Node，当它想具备组成双向链表的能力时，就需要 Node 去继承 LinkedHashMap.Entry，这个时候才是得不偿失的。  

# 插入元素
由于继承自 HashMap，因此插入的逻辑大部分是一样的。  

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 回调方法，在要添加的键值对存在，并且新值进行了覆盖的情况下调用
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    // 回调方法，新节点插入后调用
    afterNodeInsertion(evict);
    return null;
}

// 这是 HashMap 创建新节点的逻辑
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}

// 这是 LinkedHashMap 创建新节点的逻辑
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}
// 链接链表的最后一个节点
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    // 保存尾节点
    LinkedHashMap.Entry<K,V> last = tail;
    // 修改尾节点为新节点
    tail = p;
    // 如果尾节点为空则表示是第一个节点
    if (last == null)
        head = p;
    else {
        // 链接新节点的上一个节点，即之前的尾节点
        p.before = last;
        // 之前的尾节点的下一个节点为当前节点
        last.after = p;
    }
}
```

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
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
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
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            // 删除节点后的处理方法
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
// HashMap 的实现
void afterNodeRemoval(Node<K,V> p) { }

// LinkedHashMap 的实现
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    // 置空
    p.before = p.after = null;
    // 如果没有前置节点，则表示该节点就是第一个节点
    if (b == null)
        head = a;
    else
        // 将该节点的前置节点的 after 置为该节点的后置节点
        b.after = a;
    // 如果没有后置节点，则表示该节点就是尾节点
    if (a == null)
        tail = b;
    else
        // 将该节点的后置节点的 before 置为该节点的前置节点
        a.before = b;
}
```

# 访问顺序
LinkedHashMap 中维护了一个成员变量 `accessOrder`，默认值为 false，可以在调用构造方法时指定。当它为 true 时，LinkedHashMap 将会按照元素访问的顺序来维护链表；当它为 false 时，会按照元素插入（添加）顺序维护链表。按照元素访问顺序维护链表其实就是在我们调用 `get()`、`getOrDefault()` 等方法时将这些方法访问的节点移动到链表的尾部即可。  

```java
public V get(Object key) {
    Node<K,V> e;
    // 调用 HashMap 的 getNode() 方法
    if ((e = getNode(hash(key), key)) == null)
        return null;
    // 如果为 true，移动当前访问的元素到链表尾部
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}

void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    // 当前访问的节点不是尾节点才继续执行下面的逻辑
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        // 将当前节点的后置节点置空
        p.after = null;
        // 如果当前节点的 before 为空，表示当前节点就是头节点
        if (b == null)
            head = a;
        else
            // 将当前节点的前置节点的 after 置为当前节点的后置节点
            b.after = a;
        // 当前节点的后置节点不为空，则将当前节点的后置节点的 before 置为
        // 当前节点的前置节点
        if (a != null)
            a.before = b;
        else
            // 存疑！
            // 当前节点后置节点为空，表示当前节点是尾节点
            // 将 last 指定为尾节点的前一个节点
            last = b;
        if (last == null)
            head = p;
        // 前置节点不为空
        else {
            // 将当前节点的 before 链接到尾节点的上一个节点
            p.before = last;
            // 尾节点的上一个节点的 after 链接为当前节点
            last.after = p;
        }
        // 将尾节点置为当前节点
        tail = p;
        ++modCount;
    }
}
```

遍历的效果如下：  

```java
public static void main(String[] args) {
    Map<String, String> map = new LinkedHashMap<>(16, 0.75f, true);

    map.put("China", "Beijing");
    map.put("India", "Delhi");
    map.put("Korea", "Seoul");
    map.put("Japan", "Tokyo");

    map.get("China");

    map.forEach((k, v) -> System.out.println("key=" + k + " value=" + v));
}
```
		
```
key=India value=Delhi
key=Korea value=Seoul
key=Japan value=Tokyo
key=China value=Beijing
```

# LRU 缓存实现
在插入元素的代码中有一段执行了一个回调方法 `afterNodeInsertion()`，实现的是在元素插入后执行的操作，该方法在 LinkedHashMap 中的实现为：  

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        // 删除头节点
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

// 可以在继承 LinkedHashMap 时重写该方法
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

当 `accessOrder` 设置为 true 时，LinkedHashMap 中的头节点就变成了最近最少使用的节点，`afterNodeInsertion()` 方法删除的就是头节点，因此可以通过继承 LinkedHashMap 来实现一个简单的 LRU 缓存。在重写 `removeEldestEntry()` 方法时，可以根据当前节点的数量来决定是否要删除最近最少使用的节点，也可以根据节点的存活时间等其他方式来决定。  

下面是通过继承 LinkedHashMap 实现的一个简单的 LRU 策略的缓存。  

```java
public class SimpleCache<K, V> extends LinkedHashMap<K, V> {

    /**
     * 限制最大元素个数
     */
    private int limit;

    public SimpleCache(int limit) {
        super(limit, 0.75F, true);
        this.limit = limit;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > limit;
    }
}
```
