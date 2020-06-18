---
title: 动静分离与 AJP 协议
date: 2020/6/18 20:37:0
tags: [架构]
categories: [架构]
---

像 Tomcat、Jetty、Undertow 等这些 Web 容器（也可以看作是 Servlet 容器），它们既可以运行动态程序（Servlet/JSP），又可以处理静态资源，但是一般情况下，它们处理静态资源的能力与 Apache、Nginx 等这些 HTTP Server 相比要弱很多，因此在很多系统中会采用动静分离的设计，即使用 HTTP Server 作为入口网关，所有的静态资源由 HTTP Server 直接处理，而其他的动态内容则转发给 Servlet 容器。

<!--more-->

使用 Apache HTTP Server 作为入口网关，与 Servlet 容器建立关系一般有两种方式：HTTP 代理和 AJP 代理。利用 Apache 自带的 `mod_proxy` 和 `mod_proxy_http` 模块配合，就可以实现 HTTP 代理。而要实现 AJP 代理，可以使用 Apache 自带的 `mod_proxy` 和 `mod_proxy_ajp` 模块配合，或者使用 `mod_jk` 模块，JK 模块稳定性强，但是配置较为复杂。

下面需要多说一下 AJP 协议，也就是 Apache JServ Protocol version 1.3（简称 ajp13）。AJP13 是一个定向包（面向数据包的）协议，出于性能考虑，它采用了更具可读性的纯文本的二进制格式传输数据。**通过该协议，HTTP Server 能够直接与 Servlet 容器进行通信**。实现了该协议的 HTTP Server 会与 Servlet 容器建立 TCP 连接，为了减少进程反复创建 Socket 造成的资源消耗，它们之间会尝试保持持久性的 TCP 连接（长连接），一个 HTTP Server 一般会与一个 Servlet 容器维持多个 TCP 连接（TCP 连接池）。对于多个请求/响应会重用一个连接，一旦连接分配给一个特定的请求，在请求处理周期结束之前该连接将不再响应其他任何请求。换句话说，请求不会通过连接进行多路复用，这么做可以使连接两端的编码变得简单，但是可能会导致某一时刻同时打开大量的连接。

![AJP](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202006190013/2020/06/18/Aa1.png)

使用 HTTP 代理只能保持短时间内的连接，可能需要经常进行 TCP 的三次握手和四次挥手，而使用 AJP 代理可以长时间保持 TCP 连接，理论上来说性能更好。但是在目前 Nginx 为主流的大环境下，AJP 几乎没有应用环境（Nginx 官方不支持 AJP 协议），因此我们在使用 Tomcat 的时候一般会选择关闭 AJP 连接器。