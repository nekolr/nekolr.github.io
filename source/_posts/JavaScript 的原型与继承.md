---
title: JavaScript 的原型与继承
date: 2017/9/14 17:45:0
tags: [JavaScript]
categories: [JavaScript]
---
预备知识点：
		
(1) 在 JavaScript 中，除了字符串、数字、true、false、null 和 undefined ，其他都是对象。
		
(2) 在 JavaScript 中，每个对象都有一个原型指针，指向该对象所继承的原型对象。该对象仅供 JavaScript 引擎内部使用，一般我们无法直接使用它，也最好不要使用它。但是在一些浏览器中，可以使用对象实例的 `__proto__` 属性，可以认为它就是那个原型指针。
		
<!--more-->
		

(3) 在 JavaScript 中，每个函数都有一个 `prototype`属性和一个原型指针，如果一定要使用该指针，可以通过 new 和构造函数调用来创建一个对象实例，然后通过对象实例来调用 `__proto__` 属性，也可以直接通过 `函数.prototype.__proto__` 来调用。
		
有了这些了解，可以正式开始学习 JavaScript 的原型与继承了。
		
## 函数创建过程
首先写一个函数字面量：
		
![函数字面量 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/eL.png)
		
函数 fn 除了 name 等属性外，还包含一个 prototype 属性和一个原型指针。
		
其中 prototype 属性包含一个 constructor（指向函数 fn 本身）和一个指向 Object.prototype 的原型指针。
		
函数 fn 的原型指针指向 Function.prototype
		
![构造函数指向函数本身 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/Jj.png)
		
![函数的结构 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/6L.png)
		
用更直观的图来表示就是：
		
![函数的结构_图 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/mK.png)
		
## 构造函数
按照 ECMA 的定义：
*Constructor is a function that creates and initializes the newly created object.*
构造函数是一个创建并初始化一个新对象的函数
		
结论是：任何一个函数都可以是一个构造函数。
		
## 原型
函数在创建的时候会自动添加一个 prototype 属性，这个属性就是函数的原型。
		
通过 new 关键字和构造函数创建的对象的原型，就是构造函数的 prototype 属性指向的那个原型对象。
		
![prototype](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/q0.png)
		
上面的代码，反映在图中：
		
![prototype_图 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/v8.png)
		
那么原型有什么用呢？在这之前，先了解一下 new 运算符。
		
![new](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/7l.png)
		
![new 结构 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/jP.png)
		
通过使用 new 运算符和构造函数调用，能够创建一个新对象。那为什么不直接使用对象字面量的方式（var obj = {}）呢？通过上图能够发现，通过对象字面量创建的对象继承自 Object.prototype，而通过 new 和构造函数调用创建的对象继承自 Fn.prototype
		
![new1](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/QV.png)
		
其实这个 new 操作，大体上可以分为三步：
		
step 1 ：新建一个对象并赋值给 fn 。
var fn = {}
		
step 2 ：改变该对象的原型指针，指向函数 Fn 的原型。
fn.[[Prototype]] = Fn.prototype
		
step 3：调用函数 Fn ，同时把 this 指向对象 fn，对对象进行初始化。
Fn.apply(fn, arguments)
		
## 原型链
真正体现原型作用的是原型链。
		
JavaScript 中所有的对象都有一个 [[Prototype]] 属性，保存着对象所继承的原型，由 JS 编译器在对象创建时自动添加。
		
对象在查找某个属性时，会首先遍历自身的属性，如果没有则会继续查找 [[Prototype]] 所引用的对象的属性，如果没有则继续查找 [[Prototype]].[[Prototype]] ，以此类推，直到 [[Prototype]]...[[Prototype]] 为 undefined （Object 的 [[Prototype]] 就是 undefined）
		
## 继承
有了原型链，就可以进行继承了。
		
写一个函数字面量，则这个函数对象的原型指向 Object.prototype ，如何让这个函数对象的原型指向 另一个函数对象的原型呢？
		
### 让函数 A 的原型指向函数 B 的原型
方式一：使用 `__proto__`
		
![方式一 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/nM.png)
		




`__proto__ ` 不是 ECMA 的标准方法，只在某些浏览器中能够使用，如何使用标准的方法呢？
		
方式二：通过 new 和构造函数调用
		
![方式二 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/Dw.png)
		
这里隐含着，将 A.prototype.[[Prototype]] = B.prototype。但是这样做会产生一个问题，就是 A.prototype.constructor 为 undefined。
		
![方式二结构 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/Mp.png)
		
此时需要再将 A.prototype.constructor 重新赋值回去。
		
![方式二 1](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/BO.png)
		
![方式二结构 1](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/wX.png)
		
## 重写原型
		
![重写原型 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/3n.png)
		
**总结：**
		
每个对象都有一个原型指针（`__proto__`），指向该对象所继承的原型对象。
		
每个函数都有一个原型指针（`__proto__`）和一个 prototype 属性。
		
使用函数和 new 来创建新对象（var fn = new Fn()），隐含了三步操作。
step 1：var fn = {}
step 2：fn.[[Prototype]] = Fn.prototype
step 3：Fn.apply(fn, arguments)
		
## 参考
> [JavaScript 原型和继承 ](http://blog.jobbole.com/19795/)  

> [JavaScript 原型中的哲学思想 ](https://segmentfault.com/a/1190000005824449)  

> 《JavaScript 权威指南》