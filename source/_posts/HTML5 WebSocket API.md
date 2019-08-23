---
title: HTML5 WebSocket API
date: 2018/1/6 10:49:0
tags: [WebSocket,HTML5]
categories: [WebSocket]
---
HTML5 规范提供了 WebSocket API，使 Web 页面能够使用 WebSocket 协议与远程主机进行双向通信。		
		
<!--more-->
		
# 构造函数
```js
WebSocket WebSocket(in DOMString url, in optional DOMString protocols);
WebSocket WebSocket(in DOMString url,in optional DOMString[] protocols);
```
- url
表示要连接的响应 WebSocket 的地址。
- protocols
[可选 ] 这些字符串用来表示子协议，这样做可以让一个服务器实现多种 WebSocket 子协议。默认为空。  

示例：  

```js
var ws = new WebSocket("ws://example.com:80/test")
```
		
# 常用属性
| 属性名 | 类型 | 描述 |
| ------------ | ------------ | ------------ |
| url | DOMString | [只读 ] 传入构造器的 URL。它必须是一个绝对地址的 URL。 |
| binaryType | DOMString | 表示被传输二进制的内容的类型。创建 WebSocket 对象时，默认赋值为 blob；在获取该属性值时，会返回最后设置的一个值，可能是 blob 或 arraybuffer。 |
| readyState | unsigned short | [只读 ] 连接的当前状态。取值是 Ready state constants 之一。 |
| onopen | EventListener | 连接打开事件的事件监听器。当 readyState 的值变为 OPEN 的时候会触发该事件。该事件表明这个连接已经准备好接受和发送数据。这个监听器会接受一个名为"open"的事件对象。 |
| onmessage | EventListener | 消息事件的事件监听器，当有消息到达的时候该事件会触发。这个 Listener 会被传入一个名为"message"的 [MessageEvent](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent) 对象。 |
| onerror | EventListener | 错误发生时监听 error 事件的事件监听器。会接受一个名为“error”的 event 对象。 |
| onclose | EventListener | 连接关闭事件监听器。当 WebSocket 对象的 readyState 状态变为 CLOSED 时会触发该事件。这个监听器会接收一个叫 close 的 [CloseEvent](https://developer.mozilla.org/zh-CN/docs/Web/API/CloseEvent) 对象。 |
		
onopen、onmessage、onerror 和 onclose 属性，除了直接给属性赋值外，还可以通过给 WebSocket 添加监听事件的方式来使用。	

```js
ws.onopen = function(event) {
	// logic
};
```
		
```js
ws.addEventListener("open", function(event){
	// logic
})
```
# Ready state 常量
| 常量 | 值 | 描述 |
| ------------ | ------------ | ------------ |
| CONNECTING | 0 | 连接还没建立 |
| OPEN | 1 | 连接已经建立并准备好数据传输 |
| CLOSING | 2 | 连接正在关闭，或已调用 close() 方法 |
| CLOSED | 3 | 连接已经关闭 |
		
# 方法
## send(data)
- data
要发送的数据。参数类型可能是 DOMString、ArrayBuffer、Blob 或 ArrayBufferView。

## close([code],[reason])
- code
[可选 ] 一个 unsigned short 数字值表示关闭连接的状态号，表示连接被关闭的原因。如果这个参数没有被指定，默认的取值是 1000 （表示正常连接关闭）。具体取值请看 CloseEvent 的 [code 属性取值 ](https://developer.mozilla.org/en-US/docs/Web/API/CloseEvent#Status_codes)。
		
- reason
[可选 ] 一个可读的字符串，表示连接被关闭的原因。这个字符串必须是不长于 123 字节的 UTF-8 文本（不是字符）。	

# 扩展实现
- [reconnecting-websocket](https://github.com/joewalnes/reconnecting-websocket)
基于 WebSocket API 封装，支持自动重连。
		
- [SockJS-client](https://github.com/sockjs/sockjs-client)
由于 WebSocket API 有浏览器版本限制（IE 10+），SockJS 为了解决这个问题，底层首先使用 WebSocket API 尝试建立连接，如果失败，会根据不同浏览器的特点选择流传输或者轮询的方式。		

# 参考		
> [MDN WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)

> [The WebSocket API](https://www.w3.org/TR/websockets/)

> [MDN Message​Event](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent)

> [MDN Close​Event](https://developer.mozilla.org/zh-CN/docs/Web/API/CloseEvent)

> [理解 DOMString、Document、FormData、Blob、File、ArrayBuffer](http://www.zhangxinxu.com/wordpress/2013/10/understand-domstring-document-formdata-blob-file-arraybuffer)  

> [Comparison of WebSocket implementations](https://en.wikipedia.org/wiki/Comparison_of_WebSocket_implementations)

> [WebSocket 教程](http://www.ruanyifeng.com/blog/2017/05/websocket.html)