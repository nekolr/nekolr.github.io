---
title: Java 集合框架
date: 2017/8/21 21:50:0
tags: [Java,Java 集合框架]
categories: [Java 集合框架]
---
Map 属不属于集合？在讨论这个问题之前，要先分清什么才是集合。
		
如果说实现了 Collection 接口的叫集合，那么显然 Map 并不属于集合；如果说集合指的是集合框架（或容器），那么 Map 也属于集合。
		
<!--more-->
		
《Java 编程思想》的 11 章中，明确表示了集合包含 List、Set、Queue 和 Map ；《Java 核心技术 卷一》中也表示集合有两个基本的接口，Collection 和 Map。
		
集合框架属于 Java 的基础知识，也是日常开发中常用的工具，面试也经常考察，因此作为一个 Java 程序员应该熟练掌握它。
		
![collection_framework](https://img.nekolr.com/images/2018/04/14/d2n.png)
		
### ArrayList
底层维护着一个 Object 数组。使用默认的构造函数会初始化一个容量为 0 的数组，在放第一个元素时会扩容成容量为 10 的数组。可指定初始化容量。扩容时会重新开辟一个计算好容量的数组，并使用 `System.arraycopy() `复制元素，每次扩容后容量为上次容量的 1.5 倍，容量上限为 `Integer.MAX_VALUE - 8（2^31-1-8）`
		
```java
add(E e);
```
在容量足够时速度很快，在容量不足时，数组会扩容，因此在可以预见的元素个数超过默认的 10 个时，最好在初始化时指定一个合适的容量。
		
```java
add(int index,E e);
remove(int index);
remove(E e);
```
这些方法都需要使用 `System.arraycopy()` 来移动元素，效率低，并且越是靠前的元素，需要移动的元素越多。
		
```java
get(int index);
set(int index,E e);
```
使用索引，速度快。
		
**总结：使用 ArrayList 最好能够指定容量，获取和修改元素较快，删除效率较低。**
		
### LinkedList
底层维护着一个类似双向链表的结构，元素类型是 LinkedList 的静态内部类 Node ，每个 Node 包含它的前一个 Node、后一个 Node 和它本身的数据。LinkedList 将它的头结点和尾节点作为成员变量，每次添加元素都会开辟一个新的 Node 节点，无容量上限。
		
```java
get(int index);
set(int index,E e);
```
查找和修改需要遍历，当要查找的索引位置超过链表大小的一半时，反向遍历。
		
```java
add(E e);//向尾部插入
addFirst(E e);
addLast(E e);
removeFirst();
removeLast();
```
在 LinkedList 中，添加方法就是向尾部添加元素。上述方法只对链表的头尾两侧操作，不需要移动指针，效率高。
		
```java
add(int index,E e);
remove(E e);
remove(int index);
```
上述方法需要遍历查找索引，之后再改变元素前后指针指向，效率相对也较高（与数组不同，不需要改变结构）。
		
**总结：使用 LinkedList 向前后插入元素效率高，其余需要知道索引位置的方法效率相对要低，但是不需要修改结构，是以空间换时间的思路。**  
