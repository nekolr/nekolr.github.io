---
title: RPC
date: 2019/11/13 11:24:0
tags: [分布式,RPC]
categories: [分布式]
---

RPC，也就是远程过程调用（Remote Procedure Call），通俗的解释就是通过网络来请求服务，而不需要了解底层网络技术的协议和细节。可能就是因为这个通俗的解释，造成了很多人混淆了 HTTP 与 RPC。

<!--more-->

首先需要明确的是 HTTP 与 RPC 从严格意义上讲不是并行的概念，虽然它们的最终目的都是通过网络来请求服务，但是 HTTP 只是一种无状态的网络协议，而 RPC 更多的是一种概念、编程模式或者说是框架。维基百科上对于 RPC 的定义更为严谨，它说：RPC 是指计算机程序在运行过程中导致子进程在不同的地址空间（通常是指不同的机器）运行时，其编码方式与在本地的过程调用相同，即无论子进程（子程序）正在执行的是本地程序还是远程程序，程序的编码基本相同。比如在 Java 中，使用 Dubbo 获取远程业务类时与在本地使用该类的方式是一样的。

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    /**
    * 此处的 OrderService 是通过 RPC 获取的
    */
    @com.alibaba.dubbo.config.annotation.Reference
    private OrderService orderService;

    @GetMapping("/{orderId}")
    public Order getOrder(@PathVariable String orderId) {
        return orderService.getOrderById(orderId);
    }
}
```

RPC 是进程间通信（IPC，Inter process communication）的一种形式，因为不同的进程具有不同的地址空间，与之相对的是本地过程调用（LPC，Local Procedure Call）。要实现一个 RPC 框架，首先需要解决的就是网络传输，也就是通信的问题。通信协议可以选择使用 HTTP，比如 Google 开源的 gRPC 就是使用的 HTTP2 作为通信协议；也可以选择在 TCP 的基础上定义自己的私有协议，比如 Dubbo 使用的 Dubbo 协议就是在 TCP 协议的基础上封装了自己的报文格式。

当我们解决了通信问题后，就可以开始调用远程方法了，那么我们怎么知道要调用哪个方法呢？在本地的时候，我们可以很容易的调用某个对象的某个方法，但是远程的应用就不行了，因为这两个应用，具体来说是这两个进程的地址空间是完全不同的。所以此时需要解决的问题是寻址问题，也就是找到具体是调用哪个主机的哪个端口的哪个方法。比如在 RMI 中，需要提供一个 RMI Registry 来注册服务的地址，通过这个地址就可以调用指定的方法。

好了，我们现在可以找到这个方法，但是这个方法不可能都是空参数的方法啊，如果方法需要传参怎么办？在本地调用时，我们只需要把参数压到栈里，然后让函数到栈里读取就可以了。但是在远程调用的过程中，客户端跟服务端是不同的进程，因此不能直接通过内存来传递。甚至有时候我们的客户端与服务端都不是使用同一种语言来实现的。由于我们的参数需要经过网络传输，所以参数最终必须转换成二进制数据，而参数是在内存中的，因此我们需要将内存中的参数序列化成我们需要的二进制的形式。

# 参考
> [远程过程调用](https://en.wikipedia.org/wiki/Remote_procedure_call)

> [谁能用通俗的语言解释一下什么是 RPC 框架？](https://www.zhihu.com/question/25536695)