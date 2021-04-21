---
title: List 删除元素
date: 2017/8/9 0:14:0
tags: [Java,Java 集合框架]
categories: [Java 集合框架]
---
List 如何删除指定的元素？这个问题，不管在工作中还是面试时都经常遇到，算是比较基础的问题，但是往往基础才能考察人，如果想要比较全面的回答，还是需要仔细阅读源码的。

<!--more-->

你首先想到的可能是并发修改异常（`ConcurrentModificationException`）这个问题，但是我们先把它放在一边，先看一道普通的笔试题：

```java
List<String> list = new ArrayList<>();
list.add("a");
list.add("b");
list.add("c");
list.add("java");
list.add("java");
list.add("f");
list.add("g");
```

如上代码，请问如何删除所有值为“java”的元素？某位同学心说这还不简单。

```java
for (int i = 0; i < list.size(); i++) {
    if ("java".equals(list.get(i))) {
        list.remove(i);
    }
}
```

有毛病吗？乍一看没毛病啊，逻辑都对啊，但是！你跑一遍试试！是不是有毛病？

```
a
b
c
java
f
g
```

为什么会这样呢？我们来看看 `remove` 方法的源码（JDK 8）：

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                            numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

这里有一个很重要的方法 `System.arraycopy`，查看方法声明：

`public static native void arraycopy(Object src, int srcPos, Object dest, int destPos, int length);`

很遗憾，它是一个本地方法，但是我们可以通过多种渠道了解到这是一个数组复制的方法，作用是将一个数组复制到另一个数组中。`src` 是源数组，`srcPos` 是要复制的起始位置，`dest` 是目标数组，`destPos` 是复制到目标数组的起始位置，`length` 是复制的数组长度。

回到源码，看看 `remove` 方法是怎么复制的：`System.arraycopy(elementData, index+1, elementData, index, numMoved)`，很容易理解吧？它将要删除的元素后面所有的元素都往前移动了。

![移动图示](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/06J.png)

注意，关键来了！遇到第一个值为“java”的元素时，索引为 **3**，此时删除索引为 **3** 的元素，所有的后续元素前移，以前索引为 **4** 的元素跑到了 **3** 的位置，但是当前遍历的位置就是 **3** ，继续执行的结果可想而知。

那么正确的代码该怎么写呢？你可能首先想到用迭代器。对，用迭代器迭代元素，同时用迭代器来删除元素可行。但是根据数组复制的这个原理，我们还有一种做法：**倒序删除**。

```java
for (int i = list.size() - 1; i >= 0; i--) {
    if ("java".equals(list.get(i))) {
        list.remove(i);
    }
}
```

笔试题看完了，来看看并发修改异常。单线程的好处理，那多线程下的如何处理？