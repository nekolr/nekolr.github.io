---
title: JavaScript 之 bind、call 和 apply
date: 2020/4/18 13:15:0
tags: [JavaScript]
categories: [JavaScript]
---
在 js 中，bind、call 和 apply 这三个方法的作用很像，但是它们还是有一些区别的。

<!--more-->

让我们先看一段简单的代码：

```js
var bar = 1

var obj = {
    bar: 2,
    foo: function() {
        console.log(this.bar)
    }
}

obj.foo() // 2
setTimeout(obj.foo, 1000) // 1
```

直接调用 obj.foo 方法时，输出的结果为 2，因为此时 foo 方法是 obj 调用的，所以 this 指向的是 obj 对象；而通过定时器的方式调用，输出的结果为 1，原因是此时 this 指向的是全局环境（window）。为了让结果符合我们的预期，我们可以将 obj.foo 放在一个匿名函数中，这样 foo 方法的调用上下文就又变成了 obj。

```js
setTimeout(function() {
    obj.foo() // 2
}, 1000)
```

或者更直接的方式是修改方法调用的上下文，此时可以使用 bind 方法。

```js
setTimeout(obj.foo.bind(obj), 1000) // 结果为 2，bind 方法将方法调用的上下文与 obj 对象绑定
```

其实 call 和 apply 方法也有类似的作用，比如我们可以这样：

```js
obj.foo.call(obj) // 2
obj.foo.apply(obj) // 2
```

但是它们有什么区别呢？首先调用 bind 方法返回的是一个函数，我们再次调用这个函数才能得到结果。

```js
var bar = 1

var obj = {
    bar: 2,
    foo: function(x, y) {
        console.log(x + y)
    }
}

// 以下两种调用方式都可以
obj.foo.bind(obj, 1, 2)() // 3
obj.foo.bind(obj)(1, 2) // 3
```

而 call 与 apply 方法的区别主要在传参。

```js
obj.foo.call(obj, 1, 2) // 3
obj.foo.apply(obj, [1, 2]) // 3
```

下面我们简单实现一个相对严谨的 bind 方法。

```js
Function.prototype.bind = function(oThis) {
    if (typeof this !== 'function') {
        throw new TypeError('Function.prototype.bind - what is trying to be bound is not callable')
    }
    
    var fToBind = this
    var fNOP = function() {}
    var args = Array.prototype.slice.call(arguments, 1) // 调用 arguments 数组的 slice 方法
    
    var fBound = function() {
        // this instanceof fBound 说明返回的 fBound 被当作 new 时的构造函数调用
        return fToBind.apply(this instanceof fBound ? this : oThis, 
        // 获取调用 fBound 时的传参
        args.concat(Array.prototype.slice.call(arguments)))
    }

    // 维护原型关系
    if (this.prototype) {
        // 当执行 Function.prototype.bind 方法时，this 为 Function.prototype
        // 因此 this.prototype 即为 Function.prototype.prototype 也就是 undefined
        fNOP.prototype = this.prototype
    }

    // 使 fBound.prototype 成为 fNOP 的实例，因此返回的 fBound 如果是作为 new 的构造函数，
    // 那么 new 生成的新对象作为 this 传入 fBound，新对象的 __proto__ 就是 fNOP 的实例
    fBound.prototype = new fNOP()
    return fBound
}
```