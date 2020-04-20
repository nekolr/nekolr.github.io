---
title: JavaScript 的原型与继承
date: 2017/9/14 17:45:0
tags: [JavaScript]
categories: [JavaScript]
---

在 JavaScript 中，除了字符串、数字、true、false、null 和 undefined ，其他都是对象。
		
每个对象都有一个原型指针（隐式原型），指向该对象所继承的原型对象。该对象仅供 js 引擎内部使用，一般我们无法直接使用它，也最好不要使用它。但是在一些浏览器中，可以使用对象实例的 `__proto__` 属性，可以认为它就是那个原型指针。

<!--more-->

每个函数都有一个 prototype（显式原型）属性和一个原型指针（连接到原型对象 Function.prototype）。

# 函数创建过程
首先写一个函数字面量：
		
```js
function fn() {}
```

函数 fn 除了 name 等属性外，还包含一个 prototype 属性和一个原型指针。其中 prototype 属性是一个对象，包含一个 constructor（指向函数 fn 本身）和一个指向 Object.prototype 的原型指针。函数 fn 的原型指针指向 Function.prototype。

```js
function fn() {}
console.log(fn.prototype.constructor === fn); // true
```

![函数的结构](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004172100/2020/04/17/5de.png)

用更直观的图来表示就是：

![函数的结构_图](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/mK.png)

# 构造函数
按照 ECMA 的定义：

> *Constructor is a function that creates and initializes the newly created object.*

构造函数是一个创建并初始化一个新对象的函数。结论是：任何一个函数都可以是一个构造函数。

# 原型
函数在创建的时候会自动添加一个 prototype 属性，这个属性就是函数的原型。

```js
function Fn(a, b) {
    this.a = a
    this.b = b
}

console.log(new Fn(1, 2).prototype === Fn.prototype) // true

// 直接给函数对象添加属性
Fn.x = 'test'

// 给构造函数添加属性
Fn.prototype.constructor.wtf = function () {
    return 'wtf'
}

// 给函数对象的原型对象添加属性
Fn.prototype.say = function () {
    return 'say'
}
```

上面的代码，反映在图中：

![prototype_图](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/v8.png)
		
那么原型有什么用呢？在这之前，先了解一下 new 运算符。

```js
function Fn(a, b) {
    this.a = a
    this.b = b
}

var obj = {}
var fn = new Fn('is a', 'is b')

console.log(obj)
console.log(fn)
```

![new 结构](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004172100/2020/04/17/xPb.png)

通过使用 new 运算符和构造函数调用，能够创建一个新对象，需要注意的是**这个对象与函数对象不同，它没有 prototype 属性**。那为什么不直接使用对象字面量的方式（var obj = {}）创建对象呢？通过上图能够发现，使用对象字面量创建的对象继承自 Object.prototype，而使用 new 和构造函数调用创建的对象继承自 Fn.prototype。

```js
function Fn(a, b) {
    this.a = a
    this.b = b
}

var obj = {}
var fn = new Fn('is a', 'is b')

console.log(Fn.prototype === fn.__proto__) // true
```

其实这个 new 操作，大体上可以分为三步：第一步是新建一个对象并赋值给 fn，即 `var fn = {}`。第二步是改变对象的原型指针，将它指向函数 Fn 的原型，即 `fn.[[Prototype]] = Fn.prototype`。第三步是调用函数 Fn，同时把 this 指向对象 fn，对对象进行初始化，即 `Fn.apply(fn, arguments)`。

# 原型链
真正体现原型作用的是原型链。

JavaScript 中所有的对象都有一个 [[Prototype]] 属性，保存着对象所继承的原型，由 JS 编译器在对象创建时自动添加。

对象在查找某个属性时，会首先遍历自身的属性，如果没有则会继续查找 [[Prototype]] 所引用的对象的属性，如果没有则继续查找 [[Prototype]].[[Prototype]] ，以此类推，直到 [[Prototype]]...[[Prototype]] 为 undefined （Object 的 [[Prototype]] 就是 undefined）。

# 继承
有了原型链，就可以进行继承了。

首先我们写一个函数字面量，则这个函数对象的原型指向 Object.prototype ，如何让这个函数对象的原型指向另一个函数对象的原型呢？方式一就是使用 `__proto__`。

```js
function A() {}
function B() {}

A.prototype.__proto__ = B.prototype

console.log(B.prototype.isPrototypeOf(A.prototype)) // true
console.log(Object.getPrototypeOf(A.prototype) === B.prototype) // true
```

`__proto__ ` 不是 ECMA 的标准方法，只在某些浏览器中能够使用，如何使用标准的方法呢？方式二就是使用 new 和构造函数调用。

```js
function A() {}
function B() {}

A.prototype = new B

console.log(B.prototype.isPrototypeOf(A.prototype)) // true
console.log(Object.getPrototypeOf(A.prototype) === B.prototype) // true
```

这里隐含着将 A.prototype.[[Prototype]] = B.prototype。但是这样做会产生一个问题，就是 A.prototype.constructor 为 undefined。

![方式二结构1](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004172100/2020/04/17/EJg.png)

此时需要再将 A.prototype.constructor 重新赋值回去。

```js
function A() {}
function B() {}

A.prototype = new B
A.prototype.constructor = A

console.log(A)
console.log(B.prototype.isPrototypeOf(A.prototype)) // true
console.log(Object.getPrototypeOf(A.prototype) === B.prototype) // true
```

![方式二结构2](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004172100/2020/04/17/kNd.png)

# 重写原型
```js
function Person(name) {
    this.name = name
}

Person.prototype.sayName = function() {
    return this.name
}

var p1 = new Person('saber')
console.log(p1.sayName()) // saber
console.log(Object.keys(Person.prototype)) // ['sayName']

// 重写原型
Person.prototype = {
    constructor: Person,
    name: 'nekolr',
    age: 23,
    sayAge: function() {
        return this.age
    },
    sayName: function() {
        return this.name
    }
}

p1 = new Person() // 隐含 p1.[[Prototype]] == Person.prototype
console.log(p1.sayAge()) // 23
console.log(p1.sayName()) // undefined
console.log(Object.keys(Person.prototype)) // ['constructor', 'name', 'age', 'sayAge', 'sayName']
```

为什么是 undefined 呢？因为 p1 = new Person() 时，执行 Person 函数，并将 this 绑定给 p1，即 Person.apply(p1, arguments)。因为 arguments 中没有值，导致 p1.name 为 undefined，所以就返回 undefined，没有再从原型链中查找。

# 总结
每个对象都有一个原型指针（`__proto__`），指向该对象所继承的原型对象。每个函数都有一个原型指针（`__proto__`）和一个 prototype 属性。使用函数和 new 来创建新对象（`var fn = new Fn()`），隐含了三步操作。

```js
// step 1
var fn = {}
// step 2
fn.[[Prototype]] = Fn.prototype
// step 3
Fn.apply(fn, arguments)
```

# 参考
> [JavaScript 原型和继承](http://blog.jobbole.com/19795/)  

> [JavaScript 原型中的哲学思想](https://segmentfault.com/a/1190000005824449)  

> 《JavaScript 权威指南》