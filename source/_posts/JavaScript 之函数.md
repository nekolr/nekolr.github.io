---
title: JavaScript 之函数
date: 2020/4/18 16:47:0
tags: [JavaScript]
categories: [JavaScript]
---
在 JavaScript 中，函数是第一等公民，它的地位与其他值（数值、布尔值、字符串等）相同。同时函数也是对象，它可以保存在变量、对象和数组中，可以作为参数传递，也可以作为结果返回，同时它还可以拥有自己的方法。

<!--more-->

# 函数声明
使用 function 关键字可以声明一个函数字面量。

```js
function foo() {
    console.log(arguments)
}
```

还可以使用将匿名函数赋值给一个变量的方式声明一个函数，这时这个匿名函数又叫做函数表达式。

```js
var foo = function() {
    console.log(arguments)
}
```

# 声明提前
JavaScript 引擎将函数名视同变量名，所以函数的声明就会像变量声明一样被提升到整个代码的顶部。

```js
foo() // 这样使用并不会报错
function foo() {}
```

表面上好像是在声明之前调用了函数 foo，但是实际上由于“变量提升”，函数 foo 已经被提升到了代码顶部，也就是在调用之前已经声明了。但是如果此时采用赋值语句来声明函数就会报错。

```js
foo()
var foo = function() {} // TypeError: undefined is not a function
```

原因是上面的代码等同于下面的形式：

```js
var foo
foo() // 此时 foo 还没有被赋值，所以为 undefined
foo = function() {}
```

# 函数的属性
我们可以声明一个函数，然后打印它的属性看看。

```js
function foo() {
    console.log(arguments)
}
console.dir(foo)
```

![函数 foo](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004182309/2020/04/18/9NP.png)

## arguments
其中 arguments 并不能显式创建，它只有在函数中才可以使用（箭头函数不能使用），使用它可以接收数目不定的实参。arguments 是一个类数组，它并非数组，但是具有数组一样的访问方式，也具有数组的 length 属性。同时它还有一个 callee 属性，该属性仅在相关的函数正在执行时才可以使用，callee 属性的初始值就是正在执行的函数，这个特性使得我们可以通过它来实现匿名的函数递归。

```js
var sum = function(num) {
    if (num === 0) {
        return 0
    } else {
        return num + arguments.callee(num - 1)
    }
}
```

callee 属性的长度是正在执行的函数形参的长度。

```js
function foo(x, y, z) {
    console.log(arguments.callee.length) // 3
}
foo(1, 2)
```

## name
name 属性存放的是函数的名称，如果通过变量赋值的方式声明函数，那么 name 属性存放的就是变量的名称。具名函数的 name 属性存放的就是函数声明时指定的名称。

## length
length 属性存放的是函数在调用时预期会传入的参数个数（必须传入的参数个数）。这里预期传入的参数不包括默认值和 rest 参数（`...变量名`）。

```js
function foo() {}
console.log(foo.length) // 0

function foo(x) {}
console.log(foo.length) // 1

function foo(x, y = undefined) {} // 有默认值
console.log(foo.length) // 1

function foo(x, y = 1, ...z) {}
console.log(foo.length) // 1
```

## caller
caller 代表的是调用函数的对象，即函数执行的环境。如果执行函数的是全局环境（window），则 caller 为 null。

```js
function foo() {
    console.log(foo.caller) // null
    console.log(arguments.callee.caller) // null
}
foo()
```

```js
function foo() {
    function bar() {
        console.log(bar.caller === foo) // true
    }
    bar()
}
foo()
```

## [[prototype]]
在 JavaScript 中，每个对象都有一个 [[prototype]] 属性，在标准中，这是一个隐藏属性。[[prototype]] 也叫做隐式原型，**它指向的是这个对象的原型，具体来说就是指向创建这个对象的函数的显式原型**。在 ES5 之前没有标准的方法来访问这个内置属性，但是大多数浏览器都支持通过 `__proto__` 属性来访问一个对象的隐式原型，在 ES5 中才有了获取这个内置属性的方法：Object.getPrototypeOf()。一个对象的隐式原型是由构造该对象的方法决定的，目前有三种常见的构造对象的方法。

> 需要注意的是，Object.prototype 比较特殊，它的隐式原型为 null。可以认为 Object 是所有对象的顶层，所有的内建对象（Array、String、Number 等）都是由 Object 创建而来。

通过对象字面量的方式构造一个对象，它的隐式原型指向 Object.prototype。

```js
var foo = {
    name: 'foo',
    age: 22
}
console.log(Object.getPrototypeOf(foo) === Object.prototype) // true
```

通过构造函数构造出来的对象，它的隐式原型指向其构造函数的 prototype 属性指向的对象。

```js
function Person() {}
var p = new Person()
console.log(Object.getPrototypeOf(p) === Person.prototype) // true
```

通过 Object.create 方法构造出来的对象，它的隐式原型指向调用 create 方法时指定的对象。

```js
var person = {
    name: 'saber',
    age: 233
}
var p = Object.create(person)
console.log(Object.getPrototypeOf(p) === person) // true
```

本质上这三种构造对象的方式可以看作是一种，也就是通过构造函数的方式创建对象。对象字面量可以认为是为了方便开发人员创建对象的语法糖，本质上还是通过以下代码来创建对象的：

```js
var foo = new Object()
foo.name = 'foo'
foo.age = 22
```

而 Object.create 方法是 ES5 才提供的方法。在 2006 年，也就是 ES5 之前，道格拉斯写了一篇题为 [Prototypal Inheritance In JavaScript](http://crockford.com/javascript/prototypal.html) 的文章。在这篇文章中，他介绍了一种实现继承的方法，这种方法并没有使用严格意义上的构造函数。他的想法是借助原型可以基于已有的对象创建新对象，同时还不用因此创建自定义类型。他给出了如下函数：

```js
function object(o) {
    function F() {}
    F.prototype = o;
    return new F();
}

// 以下是分析原理的伪代码
function object(o) {
    function F() {}
    F.prototype = o
    var f = new F()
    f.__proto__ === F.prototype // true
    // 因为 F.prototype === o
    f.__proto__ === o // true
}
```

这个方法在 ES5 之前被普遍用于实现对象的继承。

## prototype
prototype 也叫做显式原型，该属性指向函数的原型对象。不是所有的对象都有 prototype 属性，只有函数才有 prototype 属性。隐式原型和显式原型的关系是：对象的隐式原型指向创建这个对象的函数的显式原型。

![隐式原型和显式原型](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004191648/2020/04/19/Yly.png)

显式原型主要用来实现基于原型的继承和属性的共享，隐式原型则主要用来构成原型链，而原型链同样用于实现基于原型的继承，比如当我们在访问一个对象的属性时，如果在这个对象中找不到，那么就会继续沿着隐式原型继续向上查找，直到隐式原型为空为止。

# 作用域
作用域是指变量存在的范围。在 ES5 中，变量只有两种作用域，一种是全局作用域，变量在“整个程序”中都存在，另一种是函数作用域，变量只在函数内部存在。对于顶层函数来说，函数外部声明的变量就是全局变量，它可以在函数内部读取。在函数内部定义的变量外部无法读取，称为局部变量。

与全局作用域一样，函数作用域内部也存在变量提升。使用 var 声明的变量，不管在什么位置，变量的声明都会被提升到函数体的头部。同时由于函数本身也是值，所以它也有自己的作用域，它的作用域与变量一样，就是其声明时所在的作用域，与其运行时所在的作用域无关。

```js
var x = 1
var foo = function() {
    console.log(x)
}
function bar() {
    var x = 2
    foo()
}
bar() // 1
```

```js
function foo() {
    var x = 1
    function bar() {
        console.log(x)
    }
    return bar
}
var x = 2
foo()() // 1
```

# 执行上下文
执行上下文也叫调用上下文、执行栈、调用栈等。执行上下文是代码被解析和执行时所在环境的抽象。在 js 中有三种执行上下文：全局执行上下文、函数执行上下文和 eval 执行上下文。

全局执行上下文是代码首次执行时默认的环境，不在任何函数中的代码都会在全局执行上下文中执行。全局上下文的创建做了两件事：首先创建一个变量对象（Variable Object，VO），在浏览器中这个变量对象就是 window 对象。然后将 this 指针指向这个全局对象。每次调用函数时，js 解释器都会为该函数创建一个对应的执行上下文，这个执行上下文就是函数执行上下文。在运行 eval 函数时，函数中的文本代码也会有自己的执行上下文，这个上下文就是 eval 执行上下文。

当浏览器第一次加载代码时，首先会进入到一个全局执行上下文中，如果在全局代码中调用了一个函数，那么程序就会进入到这个被调用的函数中，并创建一个新的执行上下文，并将这个函数执行上下文推到执行栈的顶部。如果在当前函数中再次调用了另一个函数，那么在程序执行到此处时同样会为之创建一个新的执行上下文并再次推入执行栈顶。一旦该函数执行完毕，该函数执行上下文就会从执行栈顶部弹出，以此类推，直到程序执行再次回到全局执行上下文，继续执行。

```js
var a = "Hello World"

function first() {
    console.log('inside first function')
    second()
    console.log('again inside first function')
}

function second() {
    console.log('inside second function')
}

first()
console.log('inside global execution context')
```

![执行上下文](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202004201409/2020/04/20/MDE.jpg)

在 js 解释器内部，每次调用执行上下文都会经过两个阶段：创建阶段和执行阶段。

## 创建阶段
创建阶段是在函数被调用，但还未执行任何代码之前。在创建阶段主要做三个工作，确定 this 的值、创建词法环境组件（Lexical Environment）和创建变量环境组件（Variable Environment）。其中词法环境和变量环境是 ES5 才有的，在 ES5 之前，创建阶段的工作是确定 this 的值、创建作用域链（Scope Chain）和创建变量对象（Variable Object）。

```
ExecutionContext = {  
  ThisBinding = <this value>,  
  LexicalEnvironment = { ... },  
  VariableEnvironment = { ... },  
}
```

### 确定 this 的值
在全局执行上下文中，this 指向全局对象，在浏览器中，这个值就是 window。而在函数执行上下文中，this 的值取决于函数的调用方式。如果它被一个对象引用调用，那么 this 的值就是该对象，否则 this 的值会被设置为全局对象或 undefined（严格模式）。这是语言设计上的一个错误，因为如果设计正确，当嵌套的函数被调用时，this 应该绑定到外部函数的 this 变量上。

```js
var name = 'the window'
var obj = {
    name: 'the object',
    getName: function() {
        console.log(this.name) // the object
        return function() {
            return this.name
        }
    }
}
console.log(obj.getName()()) // the window
```

### 词法环境
在 [ES6 官方文档](http://ecma-international.org/ecma-262/6.0/)中，对于词法环境是这样描述的：

> 词法环境是一种规范类型，基于 ECMAScript 代码的词法嵌套结构来定义标识符与特定变量和函数的关联关系。词法环境由环境记录（environment record）和可能为空引用（null）的外部词法环境组成。

简单来说，词法环境是一个包含标识符和变量之间映射的结构，这里的标识符指的是变量或者函数的名称，而变量是指对实际对象（包括函数对象）或原始值的引用。词法环境有两个组成部分，环境记录和对外部环境的引用。环境记录中存储变量和函数声明的实际位置，对外部环境的引用意味着它可以访问外部的词法环境（可以简单理解为作用域链）。

与全局执行上下文和函数执行上下文对应，词法环境也有全局词法环境和函数词法环境之分。全局词法环境中，对外部环境的引用为 null，因为它本身就是最外层的环境，除此之外它还拥有一个全局对象及其关联的属性和方法（比如 Array 方法），以及任何用户自定义的全局变量。函数词法环境中，用户在函数中定义的变量被存储在环境记录中，对外部环境的引用可能是全局环境，也可能是包含内部函数的外部函数的环境。对于函数词法环境而言，环境记录还包含了一个 arguments 对象。

环境记录也有两种类型，声明性环境记录和对象环境记录。其中函数词法环境中的环境记录是声明性环境记录，它存储变量、函数和参数。在全局词法环境中的环境记录是对象环境记录，它存储全局执行上下文中出现的变量和函数。词法环境在伪代码中看起来就像这样：

```
// 全局环境
GlobalExecutionContext = {
    // 全局词法环境
    LexicalEnvironment: {
        // 环境记录
        EnvironmentRecord: {
            Type: "Object", // 类型为对象环境记录
            // 标识符绑定在这里 
        },
        outer: < null >
    }
}
// 函数环境
FunctionExecutionContext = {
    // 函数词法环境
    LexicalEnvironment: {
        // 环境纪录
        EnvironmentRecord: {
            Type: "Declarative", // 类型为声明性环境记录
            // 标识符绑定在这里 
        },
        outer: < Global or outerfunction environment reference >
    }
}
```

### 变量环境
变量环境也是一个词法环境，它具有词法环境的所有属性，包括环境记录和对外部环境的引用。在 ES6 中，变量环境和词法环境的唯一区别就是：词法环境用于存储函数声明和变量（let 和 const）绑定，而变量环境仅用于存储变量（var）绑定。举个例子：

```js
let a = 20
const b = 30
var c

function multiply(e, f) {
    var g = 20
    return e * f * g
}
c = multiply(20, 30)
```

```
// 全局执行上下文
GlobalExecutionContext = {
    // this 绑定为全局对象
    ThisBinding: <Global Object>,
    // 词法环境
    LexicalEnvironment: {
        //环境记录
        EnvironmentRecord: {
            Type: "Object",  // 对象环境记录
            // 标识符绑定在这里
            // let const 创建的变量 a b
            a: < uninitialized >,
            b: < uninitialized >,
            multiply: < func >
        }
        // 全局环境外部环境引入为 null
        outer: <null>
    },
    // 变量环境
    VariableEnvironment: {
        EnvironmentRecord: {
            Type: "Object",  // 对象环境记录
            // 标识符绑定在这里
            // var 创建的 c
            c: undefined,
        }
        // 全局环境外部环境引入为 null
        outer: <null>
    }
}

// 函数执行上下文
FunctionExecutionContext = {
    // 由于函数是默认调用 this 绑定同样是全局对象
    ThisBinding: <Global Object>,
    // 词法环境
    LexicalEnvironment: {
        EnvironmentRecord: {
            Type: "Declarative",  // 声明性环境记录
            // 标识符绑定在这里
            // arguments 对象
            Arguments: { 0: 20, 1: 30, length: 2 },
        },
        // 外部环境引入记录为 </Global>
        outer: <GlobalEnvironment>
    },
    // 变量环境
    VariableEnvironment: {
        EnvironmentRecord: {
            Type: "Declarative",  // 声明性环境记录
            // 标识符绑定在这里
            // var 创建的 g
            g: undefined
        },
        // 外部环境引入记录为 </Global>
        outer: <GlobalEnvironment>
    }
}
```

## 执行阶段
在执行阶段，js 引擎会完成对所有变量的分配，最后执行代码。如果在声明的实际位置找不到 let 变量的值，那么会分配 undefined 值给它。

## 拓展思考
变量对象和活动对象是 ES3 提出的老概念，其实变量对象和活动对象都是变量对象，在全局执行上下文中的变量对象允许直接使用，而在函数执行上下文中的变量对象是不能直接访问的，此时由激活对象（Activation Object）扮演 VO 的角色，我们可以在函数执行上下文中直接使用 AO。这似乎与全局词法记录和函数词法记录相对应。

使用词法环境和变量环境可以比较清楚地解释为什么 var 存在变量提升，而 let 和 const 却不会。

```js
console.log(x) // undefined
var x = 2333

console.log(y) // ReferenceError: Cannot access 'y' before initialization
let y = 233
```

# 闭包
我们知道，js 有两种作用域，全局作用域和函数作用域。在函数内部可以直接使用全局作用域中的变量，而在全局作用域中则不能使用函数作用域中的变量，有一个变通的方式可以将这种不可能变为可能，那就是闭包。

```js
function foo() {
    var x = 2333
    function bar() {
        return x
    }
    return bar
}
console.log(foo()())
```

上面的例子中，函数 bar 就是一个闭包。由于在 js 中，只有函数内部的子函数才可以读取函数内部的变量，因此可以将 js 中的闭包理解为“定义在一个函数内部的函数”。闭包最大的特点就是它能够记住诞生的环境，比如函数 bar 记住了它诞生的环境 foo，所以可以通过 bar 得到 foo 内部的变量。

闭包的用途主要有两个，其中一个是用于读取函数内部的变量，这个特性可以方便我们实现封装（封装私有属性，对外提供公有方法，从而保护函数内部的属性）。另一个就是可以让一些变量始终保持在内存中，即闭包可以使得它的诞生环境一直存在，因为这个原因，滥用闭包是很危险的。

# 立即执行函数
立即执行函数（Immediately-Invoked Function Expression，IIFE）能让函数在声明完成后立即执行。

```js
(function() {
    // your code
})()
```

# 参考
> [js 中 `__proto__` 和 prototype 的区别和关系？](https://www.zhihu.com/question/34183746)

> [JavaScript 教程](https://wangdoc.com/javascript/types/function.html)

> [【译】理解 Javascript 执行上下文和执行栈](https://zhuanlan.zhihu.com/p/48590085)