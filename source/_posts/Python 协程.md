---
title: Python 协程
date: 2019/2/28 10:07:0
tags: [Python,协程]
categories: [Python]
---

在学习 Python 的协程之前，先要了解 Python 的 List Comprehensions 和 Generators。  

<!--more-->  

# List Comprehensions
List Comprehensions 一般被称为列表生成式，使用它可以用来创建 list 集合。  

生成简单的 list，比如 [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]，可以使用 `list(range(1, 11))` 来实现。对于复杂的，比如 `[1*1, 2*2, 3*3, ..., 10*10]`，我们就可以通过列表生成式来生成，写成 `[x * x for  x in range(1, 11)]`。  

在列表生成式中还可以加入条件判断，满足条件的才会生成，比如：`[x * x for x in range(1, 11) if x % 2 == 0]` 会仅生成偶数的平方 list 集合。  

由于 for 循环其实可以同时使用两个甚至多个变量，比如 dict 的 items() 方法可以同时迭代 key 和 value，所以列表生成式也可以使用两个变量来生成 list，比如：`[k + "=" + v for k, v in {'x': 'X', 'y': 'Y'}.items()]`。  

# Generators
使用列表生成式可以很方便地创建一个列表，但是这个列表的容量肯定是受限的，不能无限大，并且如果只使用列表前面的一部分数据，那么分配的大部分内存空间就被浪费掉了。Python 的生成器采用了另外一种思路，即并不预先创建出所有的数据，而是只将创建元素的逻辑提前编写好，在需要使用元素时，通过该逻辑计算出元素的值。  

在 Python 中，创建 Generators 主要有两种方式。  

第一种方式很简单，将列表生成式的方括号改为圆括号即可。  

```python
# 列表生成式
l = [x * x for x in range(1, 11)]
# 生成器
g = (x * x for x in range(1, 11))
# 获取生成器的下一个值
n = next(g)
# 迭代
for n in g:
    print(n)
```

当推算的算法比较复杂时，可以用第二种方式，也就是函数的方式去实现。  

比如斐波那契数列，第一个数和第二个数都是 1，除了它俩，其余任意一个数都可由前两个数相加得到。  

```python
def fib(num):
    i, a, b = 0, 0, 1
    while i < num:
        print(b, end=' ')
        a, b = b, a + b
        i = i + 1
```

这种由前面的元素推算后续元素的逻辑非常类似生成器，此时只要将函数中的打印值改为 `yield b` 即可将函数变成生成器。  

```python
import types

def fib(num):
    i, a, b = 0, 0, 1
    while i < num:
        yield b
        a, b = b, a + b
        i = i + 1

# <class 'generator'>
print(type(fib(11)))
# True
print(isinstance(fib(11), types.GeneratorType))
```

在函数中，代码是顺序执行的，遇到 return 语句或者执行完毕后返回；而生成器在每次调用 next() 时执行，遇到 yield 语句返回，下次执行时会从上次返回的 yield 语句处继续执行。  

# 什么是协程
协程也叫做微线程，纤程，英文名为 Coroutine，可以看做是**用户态的轻量级线程。**  

在说协程之前，我们先简单回忆一下进程和线程。进程可以简单理解为程序的启动实例，比如运行一个游戏，打开一个软件，就是开启了一个进程。而线程通常从属于进程，是程序的实际执行者。**对于操作系统来说，线程是最小的执行单元，进程是最小的资源管理单元。**  

协程在外观上看就是一段函数，但是在执行过程中，函数内部是可以中断的，然后转而去执行别的函数（并不是函数调用），并在适当的时候再返回来接着执行。从现象上看，有点类似 CPU 的中断，比如：  

```python
def printNum():
    print(1)
    print(2)
    print(3)

def printWord():
    print('a')
    print('b')
    print('c')

printNum()
printWord()
```

假如上述代码通过协程去执行，那么 printNum() 在执行过程中随时可以中断去执行 printWord()，结果可能为：  

```
1
2
a
b
3
c
```

这个结果看起来像是使用了多线程，然而实际上**协程使用的是单个线程。**  

# 协程与进程和线程的区别
协程与进程和线程的根本区别是，进程和线程是操作系统级别的，而协程是编译器级别的。虽然进程和线程在语言层次都有体现，但本质上都是操作系统提供的 API，程序的执行依赖操作系统的调度算法，每次暂停的位置是不确定的，也就意味着重新开始的位置不可预知；而协程是编译器的魔术，通常插入相关的代码使得代码段能够实现分段式的执行，每次执行遇到 yield 返回，重新开始时会从上一次的 yield 处继续向下执行。  

# 协程的优势
- 不存在线程切换开销  
因为协程是函数切换而不是线程切换，**切换的位置可以由程序自身控制**，因此也就没有线程切换的开销。和多线程比，线程数量越多，协程的性能优势就越明显。  

- 不存在线程安全问题  
由于不是多线程，也就不需要多线程的锁机制，因为只有一个线程，也就不存在同时写变量冲突，在协程中控制共享资源不加锁，只需要判断状态就好了，所以执行效率比多线程高很多。  

# 协程的劣势
- 无法发挥多核性能  
由于协程使用的是单个线程，所以它无法充分发挥 CPU 的多核性能。当然我们可以通过使用多进程 + 协程的方式来充分利用多核，我们日常编写的大部分代码都没有必要这么做，除非是在 CPU 密集型应用中。  

- 协程阻塞时会阻塞整个程序  
协程在遇到阻塞操作，如 IO 操作时会阻塞掉整个程序，因此我们使用协程时一般会配合 Event Loop 进行异步操作。  

# Python 的实现
在 Python 2.x 中，协程主要通过生成器配合 yield 和 send 来实现，或者使用第三方库，比如 `greenlet`，甚至是 `gevent`。  

在 Python 3.3 中，官方加入了 yield from 语法，在 3.4 版本中还提供了支持异步 IO 的标准库 `asyncio`，在 3.5 版本中，加入了基于 `asyncio` 的新语法 `async/await`。因此在 Python 3.x 中实现协程可以有多种组合方式，比如：  

- 在 Python 3.4 版本中，可以通过 `asyncio` + yield from 的方式实现
- 在 Python 3.5 版本中，可以通过 `async/await` 实现
- 使用第三方库 `gevent`

协程在运行的过程中共有四种状态，`GEN_CREATE`（等待执行）、`GEN_RUNNING`（正在执行）、`GEN_SUSPENDED`（中断执行） 和 `GEN_CLOSED`（执行结束）。这个状态可以通过 getgeneratorstate() 方法获取，在调用生成器生成对象后，此时的协程处于 `GEN_CREATE` 状态，即等待开始的状态。接下来需要预激协程，此时可以通过 next() 或者 send(None) 预激。预激之后的协程开始执行，当运行到 yield 处会中断执行，此时的状态就是 `GEN_SUSPENDED`。  

## 基于生成器实现
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


def simple_coroutine():
        print('coroutine started')
        x = yield
        print('coroutine received: ', x)


if __name__ == '__main__':
    from inspect import getgeneratorstate

    m = simple_coroutine()
    # 获取协程的状态
    print(getgeneratorstate(m))
    # 使用 next 方法预激
    print(next(m))
    # 发送数据
    m.send(12)
```

在使用 Generators 实现协程时，yield 通常出现在表达式的右边，比如 `x = yield`。yield 可以产出值，也可以不产出值。如果 yield 右边没有表达式，那么值为 None，比如 `x = yield` 产出的值就是 None，而 `x = yield b` 产出的值为 b 的值。同时在调用方可以通过 send() 方法发送数据给协程，发送的数据会成为 yield 表达式的值。  

### 终止协程和异常处理
在 Python 2.5 之后，生成器对象添加了 throw 和 close 方法，throw 方法会让生成器在暂停的 yield 表达式处抛出指定的异常，如果生成器处理了该异常，那么代码会继续执行到下一个 yield 表达式。如果没有处理该异常，或者抛出了另一个异常，那么异常会向上传递到调用方。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


class TestException(Exception):
    pass


def demo_test_throw(b):
    print('coroutine started')
    while True:
        try:
            x = yield b
        except TestException:
            print('TestException handled')
        else:
            print('coroutine received: ', x)
        print('测试 throw 后代码的执行情况')


if __name__ == '__main__':
    m = demo_test_throw(1)
    print(next(m))
    # 抛出自定义异常
    m.throw(TestException)
    print(next(m))
    """
    coroutine started
    1
    TestException handled
    测试 throw 后代码的执行情况
    coroutine received:  None
    测试 throw 后代码的执行情况
    1
    """
```

在生成器因为没有后续语句而退出时，如果没有 yield 值产生，则会抛出一个 `StopIteration` 异常。  
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


class TestException(Exception):
    pass


def demo_test_throw(b):
    print('coroutine started')
    try:
        x = yield b
    except TestException:
        print('TestException handled')
    else:
        print('coroutine received: ', x)


if __name__ == '__main__':
    m = demo_test_throw(1)
    print(next(m))
    # 抛出自定义异常
    m.throw(TestException)
    # 最终会抛出一个 StopIteration 异常
```

close 方法会让生成器在暂停的 yield 表达式处抛出 `GeneratorExit` 异常，如果生成器没有处理这个异常，或者抛出了 `StopIteration` 异常，则调用方不会报错。如果抛出了其他异常，则会向上传递给调用方。在抛出 `GeneratorExit` 异常后，生成器一定不能产生值，否则会抛出 `RuntimeError` 异常。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


def demo_test_close(b):
    print('coroutine started')
    try:
        x = yield b
    except GeneratorExit:
        print('coroutine closed')
        # 这里加入 yield 表达式会抛出 RuntimeError
    else:
        print('coroutine received: ', x)


if __name__ == '__main__':
    m = demo_test_close(1)
    print(next(m))
    m.close()
```

### 装饰器预激协程
协程如果不预激则无法使用，为了简化调用过程，我们可以使用装饰器来预激协程。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from functools import wraps


def coroutine(fn):
    """
    装饰器，预激 fn
    :param fn:
    :return:
    """
    # 把装饰器生成器函数替换成这里的 primer
    # 函数调用 primer 函数时，返回预激后的生成器
    @wraps(fn)
    def primer(*args, **kwargs):
        # 调用被装饰的函数，获取生成器对象
        generator = fn(*args, **kwargs)
        # 预激生成器
        next(generator)
        # 返回生成器
        return generator
    return primer


@coroutine
def simple_coroutine():
    print('coroutine started')
    x = yield
    print('coroutine received: ', x)


if __name__ == '__main__':
    from inspect import getgeneratorstate

    m = simple_coroutine()
    # 获取协程的状态
    print(getgeneratorstate(m))
    # 发送数据
    m.send(12)
```

### 返回值
在 Python 3 中，生成器可以返回值，在协程结束后（通过 return 正常返回，而不是主动停止），调用方可以通过捕获 `StopIteration` 异常来获取返回值。    

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


def demo_test_return():
    total = 0
    while True:
        tmp = yield
        if tmp is None:
            break
        total += tmp
    return total


if __name__ == '__main__':
    result = 0
    m = demo_test_return()
    next(m)
    m.send(10)
    try:
        m.send(None)
    except StopIteration as e:
        result = e.value
    finally:
        print(result)
```

### yield from
yield from 是从 Python 3.3 开始添加的语法，在其他语言中，类似的结构使用 `await` 关键字。在 yield from 的后面需要添加的是可迭代对象，它可以是普通的可迭代对象，也可以是迭代器，甚至是生成器。  

我们可以通过一个例子简单认识 yield 和 yield from 的区别。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


def demo_test_yield_from(*args):
    for item in args:
        for i in item:
            yield i


if __name__ == '__main__':
    s = 'AB'
    li = [1, 2, 3]
    d = {"name": "saber", "age": 22}
    g = (i for i in range(1, 5))

    generator = demo_test_yield_from(s, li, d, g)
    print(list(generator))
```

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


def demo_test_yield_from(*args):
    for item in args:
        yield from item


if __name__ == '__main__':
    s = 'AB'
    li = [1, 2, 3]
    d = {"name": "saber", "age": 22}
    g = (i for i in range(1, 5))

    generator = demo_test_yield_from(s, li, d, g)
    print(list(generator))
```

更复杂的，我们可以在 yield from 后面加入子生成器来实现生成器的嵌套。当然使用 yield 也可以完成生成器的嵌套，但是使用 yield from 可以避免自己处理很多意料不到的异常，从而专注于业务实现。  

在生成器的嵌套中，有几个重要的概念。比如，委派生成器指的是包含 yield from 表达式的生成器函数，子生成器指的是 yield from 后边添加的生成器函数，而调用方指的是调用委派生成器的客户端。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


# 子生成器
def calculate_sum():
    total = 0
    while True:
        tmp = yield
        if tmp is None:
            break
        total += tmp
    return total


# 委派生成器
def proxy_generator(results):
    while True:
        results['total'] = yield from calculate_sum()


# 调用方
def main():
    results = {}
    cal_sum = proxy_generator(results)
    # 预激
    next(cal_sum)
    cal_sum.send(10)
    cal_sum.send(22)
    cal_sum.send(14)
    cal_sum.send(None)
    print(results['total'])


if __name__ == '__main__':
    main()
```

委派生成器的作用是在调用方和子生成器之间建立一个**双向通道**。调用方可以通过 send() 方法直接发送消息给子生成器，而子生成器产生的 yield 的值也可以直接返回给调用方。  

yield from 的具体语义很难理解，我们为什么还要使用它呢？比如我们需要返回值，完全可以麻烦点通过异常的方式获取。其实 yield from 在背后为我们做了很多事，具体做了什么可以参考下面官方提供的说明。比如 `RESULT = yield from EXPR` 语句，解释器在背后做了哪些工作呢？  

```python
_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    while 1:
        try:
            _s = yield _y
        except GeneratorExit as _e:
            try:
                _m = _i.close
            except AttributeError:
                pass
            else:
                _m()
            raise _e
        except BaseException as _e:
            _x = sys.exc_info()
            try:
                _m = _i.throw
            except AttributeError:
                raise _e
            else:
                try:
                    _y = _m(*_x)
                except StopIteration as _e:
                    _r = _e.value
                    break
        else:
            try:
                if _s is None:
                    _y = next(_i)
                else:
                    _y = _i.send(_s)
            except StopIteration as _e:
                _r = _e.value
                break
RESULT = _r
```

## greenlet
使用 yield 可以实现协程，但是实现的过程不易于理解，我们可以使用第三方库 `greenlet` 来实现相同的效果。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from greenlet import greenlet
import time


def test_demo1():
    while True:
        print('inner demo 01')
        time.sleep(1)
        g2.switch()


def test_demo2():
    while True:
        print('inner demo 02')
        time.sleep(1)
        g1.switch()


if __name__ == '__main__':
    # 创建协程
    g1 = greenlet(test_demo1)
    g2 = greenlet(test_demo2)

    # 手动切换至 g1，先启动 g1
    g1.switch()
```

## asyncio
asyncio 是 Python 3.4 版本引入的标准库，它支持异步 IO 操作。asyncio 的编程模型就是一个消息循环，类似 JavaScript 中的 Event Loop，我们将需要执行的协程放入 EventLoop 中就可以实现异步操作。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import asyncio
import threading


@asyncio.coroutine
def hello_world():
    print("Hello World! (%s)" % threading.current_thread())
    # 模拟耗时操作
    r = yield from asyncio.sleep(1)
    print("Hello Python! (%s)" % threading.current_thread())


if __name__ == '__main__':
    # 获取 EventLoop
    loop = asyncio.get_event_loop()
    # 执行协程
    loop.run_until_complete(asyncio.wait([hello_world(), hello_world()]))
    loop.close()
```

`@asyncio.coroutine`把一个 generator 标记为 coroutine 类型，然后放到 EventLoop 中去执行。由于 `asyncio.sleep()` 也是一个协程，所以我们可以使用 yield from 接收，但是主线程并不会一直等待它执行完毕，而是直接中断并去执行下一个消息循环。当 `asyncio.sleep()` 返回时，主线程就可以从 yield from 处拿到返回值（此处是 None），然后继续向下执行。  

## async/await
为了更好的标识异步 IO，从 Python 3.5 开始引入了新的语法 `async` 和 `await`，使用该语法只需要将 `@asyncio.coroutine` 替换为 `async`，再把 yield from 替换为 await 即可。  

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import asyncio
import threading


async def hello_world():
    print("Hello World! (%s)" % threading.current_thread())
    # 模拟耗时操作
    r = await asyncio.sleep(1)
    print("Hello Python! (%s)" % threading.current_thread())


if __name__ == '__main__':
    # 获取 EventLoop
    loop = asyncio.get_event_loop()
    # 执行协程
    loop.run_until_complete(asyncio.wait([hello_world(), hello_world()]))
    loop.close()
```

# 参考
> [在 Unity 中 StartCoroutine/yield return 这个模式到底是怎么应用的？其中的原理是什么？](https://www.zhihu.com/question/23895384)  

> [Python 教程](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000)  

> [PEP 380 -- Syntax for Delegating to a Subgenerator](https://www.python.org/dev/peps/pep-0380/)  

> 《流畅的 Python》