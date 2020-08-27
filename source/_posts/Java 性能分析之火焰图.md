---
title: Java 性能分析之火焰图
date: 2020/8/27 12:44:0
tags: [JVM, 性能分析]
categories: [JVM]
---

在程序运行期间，可能会因为某段代码大量占用 CPU 时间而拖累整个应用的执行，能够快速定位这类热点代码的性能分析工具有很多，这里介绍的火焰图比较特殊，它可以将很多性能分析工具的采样结果以一种更加直观的方式展现出来，从而快速定位热点问题。

<!--more-->

网上很多关于火焰图的讲解最初来自于 Brendan Gregg 的博客文章：[Flame Graphs](http://www.brendangregg.com/flamegraphs.html)，火焰图的效果大致如下图：

<embed src="https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202008271727/2020/08/27/RjK.svg" width="900" height="563.4" type="image/svg+xml" />

该火焰图是一个 SVG 格式的文件，由开源工具：[FlameGraph](https://github.com/brendangregg/FlameGraph) 生成。图中的每个色块代表的是一个线程栈帧，色块中是该栈帧加载执行的函数以及一些采样信息。Y 轴代表调用栈的深度，调用从底部开始向上进行，顶部是正在执行的函数，下面是它的父函数，调用栈越深，火焰就越高。X 轴代表抽样数，如果一个函数在 X 轴占据的宽度越大，表示它被抽取到的次数越多，也就意味着它占用的 CPU 时间越长。

> 需要注意的是，图中的颜色并不重要，只用于区分不同的函数，同时 X 轴的从左到右的顺序也不重要，默认按照字典顺序排序。

我们可以简单举个例子来理解。一般在使用性能分析工具时，我们会选取一个合适的时机启动分析采样，然后等待一段时间后停止采样。假设采样时间为 20 秒，每秒采样 100 次，那么共采样 2000 次，在这 2000 次中，共有 1800 次执行的是 main 方法，其余 200 次执行的是 run 方法，同时 main 方法又调用了很多其他方法，run 方法也是，这就形成了一个从下往上的调用链，在这些调用链中，又同时会收集到每个方法采样的次数，最终形成一张火焰图。

火焰图是可以互动的，点击某个色块，该色块会被放大并占据所有的宽度，这样可以方便地查看某个函数内部的调用链信息。一般观察火焰图，就是看顶层函数占用的宽度，如果有些函数占用的宽度很大，形成了“平顶”，那么就代表该函数可能存在性能问题。当然，有时候直接观察平顶并不能快速定位问题，因为很多时候顶部的函数都是一些底层的库函数，这时候就应该先观察业务代码在火焰图中的宽度，然后再往上观察顶部的库函数来缩小范围。

IDEA 集成了 [async-profiler](https://github.com/jvm-profiling-tools/async-profiler) 和 Java Flight Recorder（JFR），可以通过它们来生成火焰图。由于 JFR 是商用工具，因此在启动 JVM 的时候需要添加解锁商业 feature 的参数：`-XX:+UnlockCommercialFeatures`。而 async-profiler 不能在 windows 平台使用，更多信息可以查看 IDEA 官方文档：<https://www.jetbrains.com/help/idea/async-profiler.html>。

# 参考
> [Java Flame Graphs](http://www.brendangregg.com/blog/2014-06-12/java-flame-graphs.html)