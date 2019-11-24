---
title: Java 方法区
date: 2018/4/17 21:29:0
tags: [Java,JVM]
categories: [JVM]
---
JVM 管理的内存可以总体划分为两部分：Heap Memory 和 Native Memory（也叫做 Native Heap），其中 Heap Memory 可以简单看作我们常说的堆内存（也叫 GC Heap），这部分内存直接受到 GC 的管理。Native Memory 也被称为 C-Heap，这部分内存是供 JVM 进程自身使用的。Heap Memory 的大小可以通过 JVM 参数设置，而 Native Memory 的可分配空间依赖于操作系统进程可分配内存的最大值。  

<!--more-->

![JVM 内存管理](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242036/2019/07/12/gzM.png)

![JVM 内存划分](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242036/2019/07/12/VAD.png)

# 方法区
在 Java 虚拟机规范中，Java 的运行时数据区域包括**虚拟机栈、本地方法栈、程序计数器、堆和方法区**，其中堆和方法区是线程共享的内存区域。方法区被用于存储已经被虚拟机加载的类信息、常量、静态变量、JIT 编译器编译后的代码等数据，同时运行时常量池也是方法区的一部分，存储类在编译器生成的各种字面量和符号引用。  

需要说明的是，Java 方法区只是一个概念，这个概念只在 JVM 规范中定义，并没有提供具体的实现细节，因此不同的虚拟机可能就有不同的实现。比如在 HotSpot 虚拟机中，在 JDK 1.7 及以前的版本中的实现是 [PermGen space（永久代）](https://plumbr.io/outofmemoryerror/permgen-space)，而在 JDK 1.8 的实现是 [Metaspace（元空间）](https://plumbr.io/outofmemoryerror/metaspace)。其他的虚拟机则不存在永久代的概念。  

## 永久代
在 JDK 1.7 以前的 HotSpot，方法区的实现是永久代。因为在物理上，方法区使用的是由 JVM 开辟的堆内存，在 JVM 启动时可以通过参数 `-XX:MaxPermSize` 来设置永久代的最大可分配内存空间。永久代的垃圾收集是和老年代（old generation）捆绑在一起的，因此无论谁满了，都会触发永久代和老年代的垃圾收集。

由于永久代主要存储类的相关信息，所以当动态生成类时会比较容易出现永久代内存溢出的情况。典型的就是 jsp 页面过多时容易出现永久代内存溢出。同时由于 JRockit JVM 没有永久代，Oracle 为了融合 HotSpot 和 JRockit，遂开始逐步移除永久代。  

JDK 对于永久代的替换移除工作从 JDK 1.7 就开始了，直到 JDK 1.8 时才完全移除。在 JDK 1.7 中，存储在永久代的部分数据就已经转移到了 Java Heap 或者是 Native Heap 了。如符号引用 `symbols` 转移到了 Native Heap，字面量 `interned strings` 转移到了 Java Heap，类的静态变量 `class statics` 转移到了 Java Heap。而到了 JDK 1.8，原本属于方法区的元数据（类数据、方法数据等）也被挪到了 Native Heap 中。至此，方法区里的数据只有 static field 还存在 GC Heap 中。  

## 元空间
元空间也是 JVM 规范中方法区的一种实现，与永久代不同的是，元空间并不在虚拟机中，使用的是本地内存，如果不显式通过虚拟机参数 `-XX:MaxMetaspaceSize` 限制元空间的大小，则它仅受本地内存大小的限制。  

**元空间在 GC Heap 之外，所以在 Java 语境中它属于 Native Memory。不过元空间实际上是由一个特殊的 GC 管理的，只是跟管理 Java 对象的 GC 不再共用一个而已。**  

## 参考

> [The Java® Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.4)

>[Presenting the Permanent Generation](https://blogs.oracle.com/jonthecollector/presenting-the-permanent-generation)

> [Java, Integration and the virtues of source](https://sourcevirtues.com/2013/01/14/java-heap-space-and-native-heap-problems/)

> [JEP 122: Remove the Permanent Generation](http://openjdk.java.net/jeps/122)

> [Java 内存分配的问题?](https://www.zhihu.com/question/35362838/answer/62443668)

> [Java PermGen 去哪里了?](http://ifeve.com/java-permgen-removed/)