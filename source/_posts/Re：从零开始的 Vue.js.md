---
title: Re:从零开始的 Vue.js
date: 2017/8/15 12:7:0
tags: [Vue.js]
categories: [Vue.js]
---
之前有大概的学习过 Vue.js，昨天想使用 Vue.js 结合服务端做一个 TodoMVC，但是发现好多东西都忘记了，准备重新学习并记录一下。

<!--more-->

# Vue 实例

## 构造器
每个 Vue.js 应用都从创建一个 Vue 实例启动的。

![vue_first](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/QnV.png)

Vue.js 实例可以看做是一个 ViewModel，因此很多人在创建实例时将它命名为 vm。

实例化时，需要传入一个 options 对象，对象包含挂载元素、数据、模板、方法、生命周期钩子等。

详细的 options 属性戳这里：[API](https://cn.vuejs.org/v2/api/#选项-数据)

## 属性和方法
每个 Vue 实例都会代理其 data 对象里的所有属性。

![proxy_data](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/qx0.png)
		
只有被代理的属性是响应的，在 Vue 实例创建后添加的新属性不具有响应特性。

除了 data 属性，Vue 实例还暴露了一些实例属性和方法（带有 $ 前缀标识）。

![vue_$](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/vp8.png)

**不要在实例属性或者回调函数中（如 vm.$watch('a', newVal => this.myMethod())）使用箭头函数。因为箭头函数绑定父级上下文，所以 this 不会像预想的一样是 Vue 实例，而且 this.myMethod 是未被定义的。**

详细的实例属性和实例方法戳这里：[API](https://cn.vuejs.org/v2/api/#实例属性)

## 实例生命周期

<img alt="life_cycle" src="https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/W03.png
" style="height:65%;width:65%;" />

Vue 实例在被创建之前要执行一系列的初始化过程，在这个过程中也可以调用一些生命周期钩子，这使得我们可以执行一些自定义的逻辑。

![life_cycle_hook](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/jzP.png)

详细的生命周期钩子戳这里：[API](https://cn.vuejs.org/v2/api/#选项-生命周期钩子)

# 模板语法
Vue.js 使用基于 HTML 的模板语法，允许声明式的将 DOM 与 Vue 实例的数据绑定。Vue.js 也支持使用渲染函数（render）来代替模板。

## 插入值

### 纯文本
使用“Mustache”语法的双大括号包裹 Vue 实例的数据，页面渲染时会被解析成纯文本。

![mustache_txt](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/BxO.png)

也可以使用指令 `v-text` 绑定元素，这样在数据更新时会更新整个元素的 `textContent`，如果只想更新部分 `textContent`，使用双大括号。

![v-text](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/e0L.png)
		
### HTML
使用指令`v-html`能够将数据解释成 HTML，但是这可能会被 XSS 攻击利用，因此只对可信数据使用 HTML 插入值，避免对用户输入的内容使用 HTML 插入值。

![v-html](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/J2j.png)

### JS 表达式
在模板中，我们可以使用简单的 JavaScript 表达式（单行表达式和三元表达式）。在模板中不应该试图访问用户定义的全局变量，而只能访问几个内建的全局变量：`Date` 和 `Math`。

![js_expression](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/nKM.png)

## 指令
指令是带有“v-”前缀的特殊属性，属性值预期是单行 JavaScript 表达式（除了 `v-for` 指令外）。**指令的作用是当表达式的值改变时响应式的作用于绑定的 DOM 元素。**

### 参数
指令可以使用“:”接收参数。如 `v-bind` 就可以接收参数（一般是属性），表示数据与 DOM 元素的属性绑定；`v-on` 也可以接收参数作为事件绑定到 DOM 元素上。

![cmline_param1](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/VO9.png)

![cmline_param2](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/gam.png)

### 后缀
指令可以使用“.”指定一个特殊的后缀，表示这个指令以特殊的方式绑定。

![cmline_teshu](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/7dl.png)

更多指令戳这里：[API](https://cn.vuejs.org/v2/api/#指令)

## 过滤器
Vue.js 允许自定义过滤器用在**双大括号**和 **`v-bind`** 中来格式化数据。过滤器要放在 JavaScript 表达式尾部，并用管道符号“|”标识。过滤器函数接收表达式的值作为第一个参数。

![filter](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/6pL.png)

### 串联
过滤器可以串联。表达式的值作为参数传给第一个过滤器函数，计算后再将结果作为参数传给第二个过滤器函数。

![chuanlian](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/2nw.png)

### 接收参数
过滤器可以接收参数。

![filter_param](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/m6K.png)

### 语法糖
有更简洁的使用 `v-bind` 和 `v-on` 语法。

![v-bind_sweet](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/D4w.png)

![v-on_sweet](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/MBp.png)

# 计算属性

## computed
模板内使用表达式很便利，但是对于复杂的逻辑最好是放在计算属性中。计算属性可能看起来和使用 `methods` 属性一样，但是区别是，使用 `computed`，在它依赖的数据没有发生改变时直接返回计算结果而不用重新计算，只有依赖的数据改变时才重新计算；使用 `methods` 每次都会调用。

![computed](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/wNX.png)

## setter
计算属性设置的方法，默认就是一个 get 方法，我们还可以添加 set 方法。

![setter](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/3gn.png)

## watch
Vue.js 提供了一个更为通用的观察 Vue 实例数据发生变化的属性 `watch`。当你希望数据变化时执行异步操作或者大量计算时很有用。

![watch](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/oo8.png)

# 引用
> <https://cn.vuejs.org/v2/guide/>