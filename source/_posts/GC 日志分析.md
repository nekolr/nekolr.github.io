---
title: GC 日志分析
date: 2020/9/3 16:41:0
tags: [JVM]
categories: [JVM]
---

阅读和分析 GC 日志是处理 Java 虚拟机内存问题的一项基础手段，GC 日志是一些人为定义的规则，每一种收集器的日志形式可能都不太相同，但是虚拟机的设计者为了方便用户阅读，将各个收集器的日志都维持了一定的共性。本文是日志分析的一般流程描述。

<!--more-->

遇到 Java 虚拟机进程的 CPU 占用率很高甚至达到百分之百，一般有两种情况：一种是大量地创建对象，导致频繁触发 GC，比如 OOM 导致频繁地 Full GC。另一种就是代码中有死循环或者接近死循环的操作。此时首先要拿到 Java 进程的进程号，我们可以 JDK 提供的工具，比如运行 `jps -l` 或者 `jcmd -l` 命令，也可以使用系统自带的命令，比如 `ps -ef | grep java`。

在拿到进程号以后，接下来就需要找出进程内占用 CPU 时间最多的线程，能够达到这个目的的命令有很多，比如使用 `top -Hp pid`，找到占用最多的那一条对应的 PID 即为线程 ID。

![top](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202009032223/2020/09/03/znY.png)

或者利用 ps 命令，比如使用 `ps -mp pid -o THREAD,tid,lwp,time`，查看占用 CPU 时间最多的线程。

![ps](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202009032223/2020/09/03/VAW.png)

接下来我们要使用 jstack 查看线程快照，这里需要注意的是，jstack 输出的线程信息包含一个 tid 和一个 nid，其中 tid 代表的是 Java 中的线程 ID，通过 `Thread.getId()` 可以得到。nid 意为 Native Thread Id，它在不同平台中的含义会有所不同，在 Linux 系统中，它代表的是线程的 pid，即 light-weight process id。由于 jstack 中的 ID 使用的是十六进制，因此我们使用 `printf "%x\n" pid` 将该线程号转换成十六进制，比如这里 13165 会被转换成 336d，然后使用 `jstack -l pid > jstack.txt` 命令将线程快照输出到文件中，接着查找 tid 为 336d 的线程信息即可。

一般到这里，基本就可以判断出是 GC 线程繁忙还是业务线程繁忙，如果是业务线程繁忙，那么就需要定位具体的代码查找原因；如果是 GC 线程繁忙，此时就可以进入下一个环节，即 GC 日志分析。

Java 虚拟机提供的关于 GC 日志的参数有很多，常用的包括：`-XX:+PrintGC`（输出 GC 日志）、`-XX:+PrintGCDetails`（输出 GC 详细日志）、`-XX:+PrintGCTimeStamps`（输出 GC 的时间戳）、`-Xloggc:./gc.log`（输出 GC 日志到 gc.log 文件）。一般出现问题的 Java 进程并没有开启输出 GC 日志，此时需要不停机的设置参数，具体来说就是通过 `jinfo` 命令来动态添加参数，比如 `jinfo -flag +PrintGCDetails pid`。

这里使用 `-XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps` 参数，可以看到具体的 GC 内容如下：

![GC 日志](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202009032223/2020/09/03/AN5.png)

最前面的是虚拟机自启动以来到当前的秒数，对应的是 GC 发生的时间。GC 日志开头的“[ GC ”说明了此次垃圾收集的停顿类型，如果有“Full”，则表示此次发生了 Stop The World。接下来的 PSYoungGen 是与 Parallel Scavenge 收集器配套的新生代的称呼。接下来方括号内，箭头前代表的是 GC 前该内存区域已使用的容量，箭头后代表的是 GC 后该内存区域已使用的容量，圆括号中代表的是该内存区域的总容量。方括号之外的，180892K->64366K(210944K) 表示“GC 前 Java 堆已经使用的容量 -> GC 后 Java 堆已使用的容量（Java 堆的总容量）”。再往后的 0.0032464 secs 表示此次 GC 的耗时，单位是秒。后面的 Times 中给出了更具体的时间信息，其中 user、sys 和 real 分别代表用户态消耗的 CPU 时间、内核态消耗的 CPU 时间、以及操作从开始到结束经过的墙上时间（Wall Clock Time）。

图中的例子可以看出，新生代在进行频繁的 GC 活动，我们也可以通过 `jstat -gc pid interval count` 命令来查看各个区的占用以及垃圾回收情况。

![jstat](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202009032223/2020/09/03/1ko.png)