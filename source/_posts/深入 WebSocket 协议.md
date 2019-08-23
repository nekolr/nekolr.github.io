---
title: 深入 WebSocket 协议
date: 2018/1/4 13:49:0
tags: [WebSocket]
categories: [WebSocket]
---
在学习 WebSocket 之前，需要先了解几个术语：		

(1) 全双工通信
也可以叫做双向同时通信，即通信的双方可以同时发送和接收消息。		

(2) 半双工通信
即通信的双方都可以收发消息，但是同一时间只能有一方进行发送，另一方进行接收。		

		
<!--more-->
		
(3) 长连接
指在一个连接上可以连续发送多个数据包，在连接保持期间，如果没有数据包发送，需要双方发链路检测包（心跳包）以确认连接的有效性。		

(4) 短连接
通讯双方有数据交互时，就建立一个连接，数据发送完成后，则断开此连接。
# 概述
WebSocket 是 IETF 提出的一个应用层协议，2008 年诞生，2011 年成为标准，HTML5 规范中给出了 WebSocket API。该协议基于 TCP 协议（为了兼容现有 HTTP 协议，使用 HTTP 发送建立连接的握手请求），它实现了客户端与服务器的全双工通信，允许服务器主动发送消息给客户端。
# 历史沿革
我们知道，HTTP 协议是基于 TCP/IP 的应用层协议，经历了 0.9、1.0、1.1 以及 2.0 版本（2015 年），其中 1.1 版本使用最广泛，这种分布式的、无状态的、基于 TCP 的请求响应式的协议为如今互联网的发展做出了巨大贡献。		
		
在 web 1.0 时代，门户网站盛行，信息汇聚集中。后来 AJAX 技术流行，互联网开始进入 web 2.0 时代，信息开始流动，web 应用开始注重交互体验，现如今，互联网更是向着 web 3.0 迈进。但是反观 HTTP 协议从 1.0 到 1.1，除了 keep-alive 和其他一些不痛不痒的改进，变化并不大。
		
传统的 web 应用与服务器交互，需要提交一个表单，服务器接收表单数据，返回一个新的页面。一般前后页面的变化并不大，这一过程其实传输了大量的冗余数据，占用带宽。AJAX 的出现解决了这个问题，同时也满足了强调用户体验的 web2.0 的初期发展的要求。但是随着互联网的发展，一些需要实时通讯的 web 应用出现，AJAX 开始显现出它的不足。
		
我们首先从协议上看，由于 AJAX 其实还是基于 HTTP 协议的，因此它本质上并没有改变 HTTP 的请求响应模型。		
![http_model](https://img.nekolr.com/images/2018/04/14/2Q2.png)		
		
客户端发送 HTTP 请求，在 HTTP 工作之前，客户端首先通过三次握手与服务器建立 TCP 连接，连接建立后，客户端请求发出，服务器接收到请求后响应。在 HTTP 1.0 版本中，这意味着一次请求响应处理完成，需要通过四次握手关闭连接。但是有时候短期内客户端可能会大量请求服务器，这个时候如果每次请求都要建立连接，可想而知是多么消耗资源和时间，因此在 HTTP 1.1 版本中，可以通过添加头信息 Connection: keep-alive 来使得通信的双方保持住连接，以便下次通信时，可以不用建立连接，节省了时间和带宽（保持连接不是完美的，在 HTTP 1.1 中，默认情况下所有的连接都会被保持，即使通信的双方不再传输数据，除非显式指定 Connection: close，连接关闭。而操作系统能够支持的最大连接数有限，如果大量的连接被保持，则后续的连接可能无法建立）。		
		
![wireshark](https://img.nekolr.com/images/2018/04/14/j0A.png)
		
![keep_alive](https://img.nekolr.com/images/2018/04/14/QOQ.png)
		
其实可以看出，即便是 HTTP 1.1，也只是增加了连接的复用，HTTP 的请求响应模型并没有发生变化，**服务器还是不能主动发送消息给客户端，客户端与服务器不能全双工通信**。		
		
另外，我们知道，HTTP 请求会在报文头部增加很多控制信息，很多时候，我们需要的数据其实很小，在使用频繁的请求时，大量无用的信息被传输，浪费了带宽。
# 轮询（Polling）
最早的处理实时 web 应用的方案，就是采用轮询（Polling，也可以称为短轮询）的方式，客户端每隔一段时间向服务器发送一次请求，以频繁的请求换来客户端与服务端数据的同步，然而并不是每次请求都能够遇到数据更新，更多的时候，请求收到的响应都是没有数据更新，这会带来无谓的网络传输，也给服务器增加了不必要的负担，是一种非常低效的实时方案。		
		
# 服务器推（Comet）
其实服务器推技术很早就已经存在了，最早是通过客户端套接口来实现服务器推，这种方式的缺点就是需要客户端安装软件，如 flash 等。随着浏览器技术的发展，在纯浏览器应用中也可以实现服务器推技术了。		
		
一种实现方式就是长轮询（Long-Polling）。客户端发起请求，服务端接收请求后，发现数据并没有更新，此时并不会做出响应，而是 hold 住连接，直到数据更新时，再做出响应。客户端收到新的数据，更新后立即发送新的请求，重复以上过程。长轮询需要注意的一点就是，连接不能长久的 hold，需要设置一个超时时间。		
		
另外一种实现方式是基于 Iframe 及 htmlfile 的流（streaming）方式，这种方式还有一个高大上的名字：The forever iframe technique，其实就是在页面隐藏一个 iframe 标签，src 属性指向一个对长连接的请求，服务器接收到请求后作出响应并一直更新连接状态（reload）以保证连接有效，响应的内容并不只是数据，而是客户端函数的调用+数据，如：`<script>js_callbk('data')</script>`。但是这种方式有一个弊端，就是浏览器的进度栏会一直显示加载未完成，为了解决这个问题，google 的技术人员使用了一个称为“htmlfile”的 ActiveX，并将该方法用到了 gmail+gtalk 中。
		

# WebSocket
不管是轮询还是 Comet，都不是真正意义上的实时，它们更多的是一种无奈的选择，但是不可否认它们在这个方向上所做的努力。		
		
如果能有一个可以双向实时通信，并且数据格式轻量的协议就好了。WebSocket 就是这样一个协议。		
		
![websocket_vs_http](https://img.nekolr.com/images/2018/04/14/qNA.png)
		
## WebSocket 协议的特点		
- 建立在 TCP 协议之上
因为 WebSocket 通信前需要通过 TCP 三次握手建立连接。
- 兼容现有 HTTP 协议
WebSocket 在握手阶段使用 HTTP 协议，默认端口也是 80 和 443。
- 数据格式较轻量
- 可以发送文本和二进制数据
- 无同源限制，客户端可以与任意服务器通信
- 标识符是 ws，如果使用加密协议，则为 wss。如 `ws://example.com:80/some/path` `wss://example.com:443/some/path`
		
## 通信过程分析
通过抓包来查看 WebSocket 的通信过程：		
		
![websocket](https://img.nekolr.com/images/2018/04/14/vDa.png)		
		
(1) 首先客户端通过 TCP 三次握手建立连接（粉色部分）。
		
(2) 客户端 WebSocket 请求发出，WebSocket 的请求头部与普通的 HTTP 请求有些区别。		
		
![websocket_header](https://img.nekolr.com/images/2018/04/14/DJ4.png)
		
先看请求头信息：		
```
GET / HTTP/1.1
Host: 121.40.165.18:8088
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
Upgrade: websocket
Origin: http://www.blue-zero.com
Sec-WebSocket-Version: 13
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.108
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Sec-WebSocket-Key: zkt8dDQa61WCBACQ0KA+pQ==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```
		
- Upgrade: websocket
表示这是一个特殊的 HTTP 请求，请求的目的就是要将客户端和服务器端的通讯协议从 HTTP 协议升级到 WebSocket 协议。
		
- Connection: Upgrade
HTTP1.1 中规定 Upgrade 只能应用在「直接连接」中，所以带有 Upgrade 头的 HTTP 1.1 消息必须含有 Connection 头。因为 Connection 头的意义就是，任何接收到此消息的人（往往是代理服务器）都要在转发此消息之前处理掉 Connection 中指定的域（不转发 Upgrade 域）。
- Sec-WebSocket-Key: zkt8dDQa61WCBACQ0KA+pQ==
这是一段浏览器随机生成的 base64 加密的密钥，server 端收到后需要提取 Sec-WebSocket-Key 信息，然后加密。
		
- Sec-WebSocket-Version: 13
客户端在握手的请求中携带的版本标识，表示这是一个升级版本，现在的浏览器都是使用的这个版本。		

再看看响应头信息：		
		
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: QbHcxHtM1yCmV3UzNEcnEbhx6Ew=
```
		
- HTTP/1.1 101 Switching Protocols
101 为服务器返回的状态码，所有非 101 的状态码都表示 handshake 并未完成。		

(3) 服务器返回状态码 101，表示连接建立成功。		
		
(4) 服务器发送消息给客户端（红色部分），客户端收到消息后，发送 ACK 给服务端表示数据接收成功。		
		
(5) 客户端发送消息给服务端（橘黄色部分），服务端接收到消息后，发送响应数据给客户端（绿色部分），客户端收到响应数据后，发送 ACK 给服务端表示数据接收成功。		

(6) 客户端发送关闭连接消息，双方经过四次握手，断开连接。		

# 参考		
> [Comet：基于 HTTP 长连接的“服务器推”技术](https://www.ibm.com/developerworks/cn/web/wa-lo-comet/)

> [WebSocket 教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)
		
> [W3C HTML5](https://www.w3.org/TR/2014/REC-html5-20141028/)

> [HTML Standard](https://html.spec.whatwg.org/multipage/infrastructure.html)

> [The WebSocket Protocol](https://tools.ietf.org/html/rfc6455)