---
title: DOM 加载与脚本执行
date: 2017/6/9 11:25:0
tags: [DOM,HTML]
categories: [JavaScript]
---
在写这篇文章之前，还在感慨前端变化之快，各种前端框架层出不穷，四年前（大一）在我刚接触 jQuery 时，觉得这就是神器啊，而现在 jQuery 在一些场景似乎已经被替代。		
		
收回思绪，回到这篇文章，首先回顾一下 DOM 文档加载的步骤。  

<!--more-->		

```
1. 解析 HTML 结构
2. 加载外部脚本和样式表文件
3. 解析并执行脚本代码
4. DOM 树构建完成 // DOMContentLoaded
5. 加载图片等外部资源
6. 页面加载完毕 // window.onload		
```
从加载顺序可以看出，JavaScript 脚本会在 DOM 树构建完成之前执行，但是如果我们的脚本在执行时使用了还未加载完毕的 DOM 对象，就会出错，所以我们一般将脚本放到 body 结束之后，html 结束之前执行，这样做是因为，一般的页面在 body 结束时，基本上 DOM 解析也就结束了，这样相当于将我们的脚本放在了上述的步骤 4 之后，步骤 5 之前来处理。或者更靠谱的是，根据两个重要步骤点及对应的事件： `DOMContentLoaded` 和 `window.onload` 来处理我们将要执行的脚本。当然，在实际的使用中，我们更多的是针对步骤 4 进行处理，因为如果针对步骤 6 来处理，需要等到外部资源加载完毕才能响应我们的脚本。  

### DOMContentLoaded		
addEventListener() 不兼容 IE9 以前的浏览器  

```js
document.addEventListener("DOMContentLoaded", function(){
	// your code
})
```
		
```js
$(document).ready(function(){
	// your code
})

$(function(){
	// your code
})
```
### window.onload
		
```js
window.addEventListener("load", function(){
	// your code
})
```
		
```js
$(window).load(function(){
	// your code
})
```