---
title: CPU Cache 与缓存一致性
date: 2022/2/7 10:33:0
tags: [操作系统]
categories: [操作系统]
---

在计算机中，存储体系是一个典型的金字塔结构，按照速度排列从上到下依次是：CPU 寄存器、CPU Cache（L1/L2/L3）、内存、SSD 固态硬盘以及 HDD 传统机械硬盘。越上层的存储设备速度越快，当然价格也更贵，容量也越小。

<!--more-->

从广义上讲，上一级的存储器都是下一级存储器的缓存。当然这里我们只关注 CPU Cache。现代 CPU 缓存通常都有三个等级，分为 L1、L2 和 L3，其中 L1 和 L2 在每个 CPU 核心中都有，而 L3 则是所有核心共享的。在 L1 高速缓存中，指令和数据是分开存储的；而在 L2 和 L3 中则不区分，称为统一缓存（unified cache）。

![CPU Cache]()

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

由于 CPU Cache 与内存容量上的差异，必然需要某种映射规则，来确定 Cache Line 与内存地址的关系，这种关系就是我们所说的映射策略，一般常见的策略有：直接映射、全相连和组相连。