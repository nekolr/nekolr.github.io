---
title: Web Server 和 Application Server
date: 2017/8/1 9:26:0
tags: [Web Server,Application Server]
categories: [其他]
---
# Web Server
Web Server（Web 服务器）是一种运行在服务器上的软件，它通过绑定服务器上的 IP 地址并监听某个 TCP 端口来接收 HTTP 请求，并将文本文件作为 HTTP 响应返回给客户端。

<!--more-->

我们常说的 HTTP Server 可以理解为就是 Web Server。一般的 Web Server 默认不支持生成动态页面文件，可以通过引入其他模块（Shell、PHP、Python 脚本程序）来支持。典型的 Web Server 有 Apache、Nginx、IIS 等。

apache 官网的说明：

> The Apache HTTP Server ("httpd") was launched in 1995 and it has been the most popular web server on the Internet since April 1996.
		
# Application Server
Application Server（应用服务器）在 Java 领域是指具备处理 HTTP 请求的能力，支持 JavaEE 技术比如 JMS、DI、JPA、Transactions、Concurrency 等，同时包含了 Web Container 的应用。

![app_server](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/8Ry.png)

与 Web Server 只提供简单的静态页面文件访问相比，App Server 为客户端提供访问业务逻辑的接口，客户端可以通过 HTTP 协议或其他协议来调用逻辑、获取返回的结果。典型的 App Server 有 WebLogic、WebSphere 等。

# 总结
实际使用中，谁扮演了某个角色，谁就可以叫这个名字。比如我使用 WebLogic 作为 Web Server，那么它就是一个 Web Server。可以看出，其实 Tomcat 也是一种 Web Server，只不过它处理静态页面文件的能力不如 Apache，但是它可以作为 JSP 和 Servlet 的容器（Web Container）。

![tomcat](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/NAW.png)

在应对大型分布式应用时，我们可以选择使用 WebLogic、WebSphere 等 App Server，当然也完全可以搭配 Nginx 等来组合使用，如使用 Nginx + WebLogic。

![组合 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/pPL.png)

对于中小型的应用，我们可以使用 Tomcat 作为容器，配和第三方的框架（Spring、Hibernate 等）来组成一个 App Server，当然也可以搭配 Nginx 等来组合使用。

> 动静分离：使用 Nginx 反向代理分发请求，所有动态资源的请求交给 Tomcat，静态资源的请求（音视频、CSS、JS 文件等）则直接由 Nginx 返回到浏览器，这样能大大减轻 Tomcat 的压力。

> 负载均衡：当一个 Tomcat 的实例不足以处理所有的请求时，可以启动多个 Tomcat 实例进行水平扩展，Nginx 的负载均衡可以把请求通过算法分发到各个不同的实例进行处理。

# 参考
> [tomcat 与 nginx，apache 的区别是什么？](https://www.zhihu.com/question/32212996)