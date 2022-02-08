---
title: CPU Cache 与缓存一致性
date: 2022/2/7 10:33:0
tags: [操作系统]
categories: [操作系统]
---

在计算机中，存储体系是一个典型的金字塔结构，按照速度排列从上到下依次是：CPU 寄存器、CPU Cache（L1/L2/L3）、内存、SSD 固态硬盘以及 HDD 传统机械硬盘。越上层的存储设备速度越快，当然价格也更贵，容量也越小。

<!--more-->

从广义上讲，上一级的存储器都是下一级存储器的缓存。当然这里我们只关注 CPU Cache。现代 CPU 缓存通常都有三个等级，分为 L1、L2 和 L3，其中 L1 和 L2 在每个 CPU 核心中都有，而 L3 则是所有核心共享的。在 L1 高速缓存中，指令和数据是分开存储的；而在 L2 和 L3 中则不区分，称为统一缓存（unified cache）。

![CPU Cache](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202202081621/2022/02/07/wEP.png)

在 Linux 系统中，我们可以通过以下命令来查看 L1/L2/L3 的大小：

```bash
# L1 数据缓存
$ cat /sys/devices/system/cpu/cpu0/cache/index0/size
32K

# L1 指令缓存
$ cat /sys/devices/system/cpu/cpu0/cache/index1/size
32K

# L2
$ cat /sys/devices/system/cpu/cpu0/cache/index2/size
256K

# L3
$ cat /sys/devices/system/cpu/cpu0/cache/index3/size
3072K
```

# CPU Cache 映射策略
CPU Cache 从内存读取到的数据是一块一块存储的，这一块可以理解为 CPU Cache 的最小缓存单位，它有一个专门的名字：Cache Line，一般它的大小为 64 Byte。在 Linux 中可以通过以下命令来查看它的大小：

```bash
$ cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
64
```

由于 CPU Cache 与内存容量上的差异，必然需要某种映射规则，来确定 Cache Line 与内存地址的关系，这种关系就是我们所说的映射策略，一般常见的策略有：直接映射、全相联和组相联。

## 直接映射
直接映射是一种多对一的映射，这种方式下，主存中的每个内存块只能有一个 Cache Line 与之对应，因此它也叫做“单路组相联”。具体来说就是使用内存块地址对 Cache Line 的个数取模。比如内存共被划分了 32 个内存块，而 CPU Cache 共有 8 个 Cache Line，假如 CPU 想要访问第 15 号内存块，如果该内存块的数据已经缓存在了 Cache Line 中，那么一定是映射在了 7 号 Cache Line 中。一般来说，缓存的索引号可以通过以下公式计算：

```
I = (Am / B) mod N
```

其中 I 为缓存索引，Am 为内存地址，B 为 Cache Line 的大小，N 为 Cache Line 的个数。Am 除以 B 是内存块的个数。

<img src="https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202202081621/2022/02/08/mgx.png" alt="直接映射" style="width: 60%" />

直接映射的优点就是结构简单，易于实现，但是存在显著的冲突问题。由于多个不同的内存块会共享同一个缓存块，一旦缓存失效则必须将缓存块当前的数据清除，这在频繁更换缓存内容时会造成大量的延迟，并且也无法有效利用程序运行期间所具有的时间局部性特征（近期访问的地址在不久的将来很有可能被再次访问）。

## 组相联
组相联是把缓存划分为多个组，每个组有若干个 Cache Line。用一句话来概括就是：组间直接映射，组内全相联。以下是一个 2 路组相联的例子：

<img src="https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202202081621/2022/02/08/nOA.png" alt="2 路组相联" style="width: 60%" />

上图将缓存分成了 s 组，每组 2 个 Cache Line，即 2 路（2 ways），主存中的每个数据块只能位于分组中的某一个，但是可以在指定分组中的任意一个 Cache Line 中。一般来说，组相联的缓存索引可以通过以下公式计算：

```
I = (Am / Nw / Na) mod N
```

其中，I 为缓存索引，Am 为内存地址，Nw 为缓存块内字数（也可以认为是 Cache Line 的大小），Na 为相联路数，N 为分组个数。

## 全相联
全相联是指主存中的数据块可能出现在任意一个 Cache Line 中，这种方式使得替换具有最大的灵活性（可以使用 LFU 或者 LRU 等算法），同时也意味着有最低的 miss 率。但是由于没有索引可以使用，检查一个 cache 是否命中需要在整个 cache 范围内搜索，这带来了查找电路的大量延时。因此只有在缓存极小的情况才有可能使用这种方式。

## 小结
其实上面的三种映射方式，其实都可以看作是组相联，直接映射是单路组相联，而全映射则只有一个分组。因此，一个主存地址映射到高速缓存大体上有三个步骤：组选择（查找 Cache Set），行匹配（查找 Cache Line）和字抽取（查找 Cache Line 中一个字的起始字节）。

比如一个 32 位系统的内存地址映射到 4 MB 高速缓存中。首先是组选择，对于直接映射来说，分组数等于 Cache Line 的个数：65536，也就意味着需要中间 16 bit 来表示 Cache Line 的编号。接下来是行匹配，对于直接映射来说，一个分组只有一个 Cache Line，不用选择。最后是字抽取，由于一般 Cache Line 的大小是 64 Byte，同时现代处理器中，存储单元一般是以字节为单位的，也是最小的寻址单元，这也就意味着一个 Cache Line 可以存储 64 个存储单元，因此内存地址的低位 6 个 bit 用于表示在 Cache Line 中的偏移量（数据从第几个字节开始）。剩余的高位 10 bit 作为内存地址的一部分，同样也会映射到 Cache Line 中，作为标记位。

<img src="https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202202081621/2022/02/08/677.png" alt="组映射" style="width: 90%" />

上图是组映射的一般表示，对于一个 E 路组相联来说，缓存被划分为了 2^s 组，即通过内存地址的中间 s 位即可找到目标 Cache Line 的对应分组。找到分组后，遍历分组中所有的 Cache Line，检查 Cache Line 中的有效位，以及对比 Cache Line 中的标记位与内存地址的高位 t bit 是否一致。当 tag 和 valid 校验成功，我们称为缓存命中，此时只需要根据内存地址的低位 b bit 计算出 Cache Line 中数据的起始字节，向后读取一个字放入 CPU 寄存器即可。

<img src="https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202202081621/2022/02/08/31e.png" alt="内存地址的一般表示" style="width: 90%" />

> 计算机各个硬件之间进行信息传递是通过贯穿整个系统的一组电子管道，称做总线，它携带信息字节并负责在各个部件间传递。硬件之间进行信息交流需要有一个统一的标准，也就是二进制信息传递规则，为了高效考虑，通常总线被设计成传送定长的字节块，也就是字（word）。字中的字节数（即字长）是一个基本的系统参数，在各个系统中的情况都不尽相同。操作系统中的 32 位（4 个字节）或 64 （8 个字节）位就叫总线的字长单位。