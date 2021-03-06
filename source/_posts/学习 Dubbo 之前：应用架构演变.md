---
title: 学习 Dubbo 之前：应用架构演变
date: 2018/5/7 14:7:0
tags: [Dubbo,架构]
categories: [架构]
---

今天在学习 Dubbo 时，看到 Dubbo 官方文档中对应用架构演变作了一个说明。

<!--more-->

![架构演变](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/07/KBb.png)

![架构演变解释](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/07/0d7.png)

这是从大的阶段划分的，下面针对每个阶段大概分析一下，不涉及过多的细节。  

# 单一应用架构演变
一个小网站，一台服务器就足够了，文件服务、数据库服务和应用都部署在一起，俗称 All In One。随着用户增多，流量增大，硬盘、CPU 和内存吃紧，一台服务器已经不能满足了。  

![单一应用架构](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/07/28e.png)

## 升级：硬件服务分离
一台服务器不行了，常规的思路就是加机器，将硬件服务分开。一台服务器部署应用，一台作为文件服务器，一台作为数据库服务器。

![硬件服务分离](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/07/Vzz.png)

## 升级：数据缓存
单机模式下，常见的性能瓶颈往往在数据库查询，也就是 ORM 的工作。常见的手段是 SQL 优化、加入缓存。SQL 优化需要专业人员根据业务热点定位，具体优化。与之相比，采用缓存一般是更直接和高效的方式，可以选择增加数据库查询缓存，当然，这也就引入了新的复杂度。

> **思考**：缓存从来不是数据库查询的特权，根据 28 原则，只要是应用中被大量访问的数据，我们都可以将它们缓存下来，然后看场景选择存放在本地（Local Cache）还是其他地方（Remote Cache）、是集中式存放还是分布式存放（Distributed Cache）。  

![缓存](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/07/gnd.png)

## 升级：读写分离、分库分表
读写分离即将对数据库的读操作和写操作分离开，读操作一个数据库，写操作另一个数据库（分成主从数据库），写操作完成后同步到另一个数据库。  

![读写分离](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/07/jqN.jpg)

当业务增长到一定规模，数据库软件管理了大量的数据库表和数据，数据库操作开始变慢，这时候通常采用的方式就是分库分表。大量的数据在数据库中一般表现为，大量的数据分散到大量的表中造成表数量巨大或者是表数量并不是很多但每张表的数据量巨大，因此分库分表大体上有两种方式：一种是垂直拆分，即把关系紧密（比如一个模块）的表拆分出来放在一个单独的数据库中；一种是水平拆分，即把表中的数据按照某种规则（例如散列）拆分到多个数据库中。当然，很多时候需要根据实际情况进行混合拆分。  

![垂直拆分](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/07/qG7.jpg)

![水平拆分](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/07/v4m.jpg)

## 升级：动静分离
将用户请求的资源分为两种：静态资源和动态资源。静态资源如：图片、js 文件、css 文件等，动态资源如：jsp、php 等需要服务器端程序处理后才返回的资源。动静分离要求将对这两种资源的请求分开处理，之所以这么分，一是可以缓解单一 Web 服务器的压力；二是可以对静态资源使用 CDN 加速。  

![动静分离](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004181107/2020/04/18/ezr.png)

## 升级：负载均衡
每台服务器的进程是有限的，我们不能无限扩展服务器的配置，因此有必要将多个服务器组合起来处理业务。通常采用负载均衡技术，根据每台服务器的状态来判断用户的请求交由哪个服务器处理。  

![负载均衡](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004181107/2020/04/18/Pra.png)

## 注意
上面的这些升级方案，很多并不是单一应用架构专属，它们很多是在架构演变的过程中产生，在一些分布式架构广泛使用。架构方案有很多共通点，我们只需要关注它们所解决的问题就可以了，有的方案并不是必须的，很多时候需要根据业务需求来决定。  

# 垂直应用架构
我们在应用的外部环境做了太多的努力，是时候关注一下应用本身了。我们的应用本身也许设计的就不合理。垂直应用架构简单来说就是将以前的单体应用拆分成几个互不相干的应用。比如人人网在之前，因为是社交网站，就把应用拆分成了 SNS，UGC 和各个游戏子应用。  

> **思考**：在垂直应用架构中，流量能够分散到各个子应用中，这从一定程度上解决了单一应用架构存在的扩容难题，但是对单体应用拆分，也就意味着很多具有相同逻辑的代码在各个子应用中无法复用。其实这很容易理解：在单体应用中，我们一般会将公共的逻辑代码提取封装，以供其他逻辑进行调用；然而到了垂直应用中，这就行不通了，我们没有办法让各个子应用去调用这个公共的逻辑部分，子应用之间相互隔离，无法交互。  

# 分布式应用架构
当我们的垂直应用越来越多，业务上的跨应用交互不可避免，此时就需要一种能够远程调用其他应用中的方法的手段。

在不考虑 RPC 之前，我们可以自己设想怎样才能在一个应用中调用另一个应用中的某个方法。

- 首先需要解决的就是网络传输，也就是**通信问题**，很明显，这两个应用之间肯定需要建立某种连接来进行通信交互。  

- 好了，我们假设两个应用已经能够互相交互了，但是我们怎么知道要调用哪个方法呢？在本地的时候，我们可以很容易的调用某个对象的某个方法，但是远程的应用就不行了，因为这两个应用，具体来说是这两个进程的地址空间是完全不同的。所以此时需要解决的问题是**寻址问题**，也就是找到具体是调用哪个主机的哪个端口的哪个方法。  

- 好了，我们可以找到这个方法了，但是不可能每次都调用空参数的方法啊。如果方法需要传参，怎么办？我们的参数需要经过网络传输，所以必须是二进制数据，而参数是在内存中的，因此我们需要将内存中的**参数序列化**成二进制的形式。  

- 远程应用收到方法的调用请求，并拿到了我们传输的参数，此时还需要将**参数反序列化**成内存的表示形式传入方法中执行，执行结束后将结果返回。返回的结果同样需要序列化传输，本地应用接收到返回的数据后再反序列化成内存中的表示方式。  

解决了以上的问题，也就解决了远程方法调用的问题，解决这些问题的实现被称为 RPC（远程过程调用）。PRC 有早期的 CORBA、Java RMI，还有基于 XML 的 XML-PRC、Web Service、SOAP，基于 JSON 的 JSON-RPC，以及 Hessian、Thrift、GRPC、hprose 等等。  

有了 PRC，分布式应用架构的主要难题也就解决了。在分布式应用架构中，将核心业务抽取出来作为单独的服务对外提供发布，并逐渐形成稳定的服务中心，各个子应用通过 RPC 去消费这些服务。  

# 流动计算应用架构
当服务越来越多，容量的评估、小服务资源的浪费、交互的越发复杂成为新的问题，此时需要增加一个调度中心基于访问压力进行实时的集群资源调度和服务治理。

# 参考
> [知乎：谁能用通俗的语言解释一下什么是 RPC 框架？](https://www.zhihu.com/question/25536695)  

> [RPC 协议的区别 ](https://segmentfault.com/q/1010000003064904?_ea=298208)