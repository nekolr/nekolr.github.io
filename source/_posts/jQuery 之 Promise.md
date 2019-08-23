---
title: jQuery 之 Promise
date: 2017/8/20 21:5:0
tags: [JavaScript]
categories: [JavaScript]
---
由于在 JavaScript 中所有的代码都是单线程执行的，因此一些耗时的操作（如网络 I/O）都采用异步的方式执行。在以前的 JavaScript 中，异步通常通过回调函数来实现，比较常见的就是 AJAX 请求成功和失败的回调函数。  

<!--more-->

```JavaScript
$.ajax({
    url: 'https://api.example.com/v1',
    type: 'get',
    success: function (response) {
        // 请求得到响应后执行该函数
    },
    error: function (error) {
        // 请求失败执行该函数
    }
})
```

但是回调函数有一个很大的缺陷就是当有多个回调嵌套调用时，很容易陷入我们常说的**回调地狱**中。比如我们可能需要发起多次 AJAX 请求，每次都需要依赖上一次 AJAX 请求返回的结果来发起下一次 AJAX 请求。  

```JavaScript
$.ajax({
    url: 'https://api.example.com/v1/saber',
    type: 'get',
    success: function (response) {
        if (response.status) {
            $.ajax({
                url: 'https://api.example.com/v1/avalon',
                type: 'get',
                success: function (response) {
                    // ...
                },
                error: function (error) {
                    // ...
                }
            })
        }
    },
    error: function (error) {
        // 请求失败执行该函数
    }
})
```

为了解决这个问题，JavaScript 社区最早提出了一种更为优雅和强大的方案，它就是 Promise。一开始有很多第三方的开源实现，包括 jQuery 自己实现的 Promise，直到 ES6 它才正式作为规范的一部分被写进了语言标准中。  

# jQuery 的 Promise
与 ES6 标准不同的是，jQuery 实现的能进行异步链式调用的对象称为 deferred（延迟）。

在使用 `$.ajax()` 时，如果在 jQuery 1.5.0 以前，它返回的是 XHR 对象，与我们平常使用的 AJAX 没有什么区别；而在更高的版本中，它返回的是一个 deferred 对象，可以进行链式操作。

```JavaScript
$.ajax("https://api.example.com/v1/saber")
    .done(function () {
        // 请求成功，相当于 success 回调方法
    })
    .fail(function () {
        // 请求失败，相当于 error 回调方法
    })
```

## 执行状态
deferred 对象有三种执行状态，分别为未完成、已完成和已失败。如果执行状态是**已完成**，则 deferred 对象会立即调用 done() 方法指定的回调函数；如果执行状态为**已失败**，则会调用 fail() 方法指定的回调函数；如果执行状态为**未完成**，则会继续等待，或者调用 progress() 方法指定的回调函数。  

## API
使用 $.deferred() 可以创建一个 deferred 对象，deferred 对象的方法主要分为两种，一种是改变执行状态的方法，另一种是状态改变时会调用的方法。  

- deferred.resolve()
可以改变 deferred 对象的运行状态为已完成。
- deferred.reject()
可以改变 deferred 对象的运行状态为已失败。
- deferred.done()
指定当运行状态为已完成时需要执行的回调函数。
- deferred.fail()
指定当运行状态为已失败时需要执行的回调函数。
- deferred.then()
指定当运行状态为已完成或者已失败时需要执行的回调函数，它可以看作是 done() 方法和 fail() 方法的结合，有时为了省事可以只使用 then() 方法。如果 then() 方法有两个参数，那么第一个参数就是 done() 方法的回调函数，第二个参数就是 fail() 方法的回调函数；如果只有一个参数，那么它等同于 done()。
- deferred.always()
不管运行状态为已完成还是已失败，它最终总是会执行。
- deferred.promise()
它的作用是在原来的 deferred 对象上返回一个新的 deferred 对象，这个新对象只开放与改变状态无关的方法，比如 done() 和 fail() 方法。

还有一个相关的方法为 $.when()，它能够为多个操作指定回调方法。  

## 指定多个回调函数
deferred 对象允许添加任意多个回调函数。  

```JavaScript
$.ajax("https://api.example.com/v1/saber")
    .done(function () {
        console.log("我是第一个成功执行");
    })
    .fail(function () {
        console.log("出错啦");
    })
    .done(function () {
        console.log("我是第二个成功执行");
    })
```

## 多个操作指定回调函数
deferred 对象允许对多个操作指定回调函数，如果都成功了，才会执行 done 方法指定的回调函数；如果有一个失败了或都失败了，就会执行 fail 方法指定的回调函数。  

```JavaScript
$.when($.ajax("https://api.example.com/v1/saber"), $.ajax("https://api.example.com/v1/avalon"))
    .fail(function () {
        console.log("出错啦");
    })
    .done(function () {
        console.log("成功执行");
    })
```

## 普通操作指定回调函数
由于 $.when() 方法的参数必须是一个 deferred 对象，因此自定义的函数最后应该返回 deferred 对象。  

```JavaScript
var $deferred = $.deferred();
function wait() {
    var task = function () {
        console.log('exec task');
        $deferred.resolve();
    };
    setTimeout(task, 1000);
    return $deferred;
}

$.when(wait()).done(function () {
    console.log('task finished');
}).fail(function () {
    console.log('task failed');
})
```

## deferred.promise()
由于 $.Deferred() 创建的对象开放了改变其状态的方法，因此我们可以从外部改变它，比如：  

```JavaScript
var $deferred = $.Deferred();
function wait() {
    var task = function () {
        console.log('exec task');
        $deferred.resolve();
    };
    setTimeout(task, 1000);
    return $deferred;
}

$.when(wait()).done(function () {
    console.log('task finished');
}).fail(function () {
    console.log('task failed');
});

// 在外部改变它的状态
$deferred.resolve();
```

为了避免这种情况，我们可以使用 deferred.promise() 方法，比如：  

```JavaScript
var $deferred = $.Deferred();
function wait() {
    var task = function () {
        console.log('exec task');
        $deferred.resolve()
    };
    setTimeout(task, 1000)
    return $deferred.promise()
}

$.when(w = wait()).done(function () {
    console.log('task finished');
}).fail(function () {
    console.log('task failed');
});

// 此时在外部改变状态无效
w.resolve();
// 但是还是可以改变 $deferred 的状态
$deferred.resolve();
```

这种写法只是将异步操作的返回值修改为了 deferred.promise()，而 deferred 对象还是暴露在全局环境下，所以更好的写法是将 deferred 封装进异步操作中。  

```JavaScript
function wait() {
    var $deferred = $.Deferred();
    var task = function () {
        console.log('exec task');
        $deferred.resolve()
    };
    setTimeout(task, 1000);
    return $deferred.promise();
}

$.when(w = wait()).done(function () {
    console.log('task finished');
}).fail(function () {
    console.log('task failed');
});

// 此时在外部改变状态无效
w.resolve();
```

$.Deferred() 还可以接收一个函数名作为参数，$.Deferred() 方法生成的 deferred 对象将作为这个函数的默认参数。因此，我们还可以这么写：  

```JavaScript
function wait($deferred) {
    var task = function () {
        console.log('exec task');
        $deferred.resolve()
    };
    setTimeout(task, 1000);
    // 此处不用返回任何值
}

$.Deferred(wait).done(function () {
    console.log('task finished');
}).fail(function () {
    console.log('task failed');
});
```
		
# 参考
> [jQuery 的 deferred 对象详解](http://www.ruanyifeng.com/blog/2011/08/a_detailed_explanation_of_jquery_deferred_object.html)

> [MDN Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)