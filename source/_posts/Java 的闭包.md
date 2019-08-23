---
title: Java 的闭包
date: 2018/4/3 15:30:0
tags: [Java]
categories: [Java]
---

不怎么严谨的说，闭包定义的要点有两个：  

> 1. 一个依赖于外部环境中自由变量的函数
> 2. 自由变量所在的外部环境必须存在

<!--more-->  

[根据 R 大的说法 ](http://rednaxelafx.iteye.com/blog/245022)，一个函数如果依赖于外部环境中的自由变量，必须保证函数依赖的自由变量的外部环境存在。这句话是重点，需要仔细理解。  

# JavaScript 的闭包

我们先看一下 JavaScript 的闭包。  

```js
function outer() {
    var p = 1;
    return function inner (param) {
        return p + param;
    }
}

var p = outer()(10); // 11
```

其中，`outer` 函数中的变量 p 就是一个自由变量，函数 `inner` 依赖于这个变量，并且在 JavaScript 中，内部函数总是可以访问其所在的外部函数中声明的变量和参数，即使外部函数已经结束（return）了，因为 JavaScript 实现了引用捕获（capture-by-reference）。  

```js
function Counter(start) {
    var count = start;
    
    return {
        increment: function() {
            count++;
        },
        
        get: function() {
            return count;
        }
    }
}

var foo = Counter(5);

foo.increment();

foo.get(); // 6
```

通过上述例子也能看出，闭包一个重要的作用就是将函数的内部变量封装，外部只能使用指定的函数来操作该变量。  

# Java 的闭包

其实在 Java 中，到处都存在闭包，只不过通常不会这么说而已。比如：  

```java
class Foo {
    private int x;
    int addWith(int y) {
        return x + y;
    }
}
```

`addWith()` 方法依赖于外部的自由变量 x ，为了让 `addWith()` 方法正确工作，它必须依附于 `Foo` 的一个实例，不然就得不到 x 的值。严格来说，`addWith()` 方法捕获的不是自由变量 x ，而是 `this`，变量 x 是通过 `this` 访问到的。  

如果这个外部环境是一个函数，并且内部函数可以作为返回值返回，那么外部环境函数中的局部变量就不能在调用结束时就销毁，**也就是说不能在栈上分配空间（每执行一个方法都会在栈上压入一个新的栈帧，栈帧包含局部变量表，方法执行完毕后，栈帧弹出销毁，所以局部变量也就被销毁了）**，如下所示：  

```js
// 这里用 JavaScript 来演示
function AddWith(x) {
    return function(y) {
        return x + y;
    }
}
```

在这个例子中，内部函数返回时，外部函数也就结束了，JavaScript 实现了引用捕获，使得内部函数在返回之后，还能够拥有外部函数环境中的自由变量而不会销毁。  

## 内部类

Java 的内部类也是一个闭包结构，如：  

```java
public class Outer {
    private int x = 100;

    private class Inner {
        private int y = 100;
        public int add() {
            return x + y;
        }
    }
}
```

Java 的内部类包含一个指向外部类的引用，因此才能够做到访问外部环境中的变量。  

## 匿名内部类

Java 的匿名内部类就比较尴尬了。比如下面的例子：  

```java
interface Inner {
    int add();
}
public class Outer {
    public Inner getInner(final int x) {
        final int y = 100;
        return new Inner() {
            int z = 100;

            @Override
            public int add() {
                return x + y + z;
            }
        }
    }
}
```

方法 `add()` 依赖于外部环境中的自由变量，此时外部方法 `getInner()` 构成了对匿名内部类的一个闭包。但是这里别扭的地方是，变量 x 和 y 都必须使用 `final` 修饰，不允许修改。当然，在 JDK 8 以后，这里已经可以不用 `final` 修饰了，但是变量 x 和 y 必须是 `effectively final` 的，即有效只读的，还是不允许修改，如下所示：  

```java
interface Inner {
    int add();
}

public class Outer {

    public Inner getInner(final int x) {
        int y = 100;
        return new Inner() {
            int z = 100;

            @Override
            public int add() {
                return x + y + z;
            }

            // Variable 'y' is accessed from within inner class,
            // needs to be final or effectively final
            public void setY() {
                y = 21; // 这里编译器报错了
            }
        };
    }
}
```

为什么会这样呢？这是因为 Java 编译器支持了闭包，但是支持的不完整。说它支持了闭包是因为编译器在编译时偷偷的对函数做了处理，把外部环境中的局部变量 x 和 y 拷贝了一份放到了匿名内部类里，如：  

```java
interface Inner {
    int add();
}
public class Outer {
    public Inner getInner(final int x) {
        final int y = 100;
        return new Inner() {
            // 编译器相当于拷贝了外部自由变量的一个副本到匿名内部类里
            int copyX = x;
            int copyY = y;

            int z = 100;

            @Override
            public int add() {
                // 这里使用的实际上是拷贝的变量
                return x + y + z;
            }
        }
    }
}
```

用 R 大的话说，就是：  

> Java 编译器实现的只是 capture-by-value，并没有实现 capture-by-reference。

从语义上看，修改 x 和 y 的值，需要反映到外部环境中的变量上，即外部环境中的变量也要变化，而实际上因为在匿名内部类中使用的其实是自由变量 x 和 y 的副本，所以在匿名内部类中修改自由变量的值是无法相应的修改外部环境中自由变量的值的，即如果修改它们的值会造成**内外环境不同步**。而如果 Java 实现了引用捕获就不会发生这个问题，所以当前的 Java 干脆设置了一个限制，就是外部环境中的变量必须使用 `final` 修饰（JDK 8 中可以不加，但是变量必须是有效只读的，还是不能修改）。  

## 其他类似的闭包结构

外部类成员方法中的内部类。  

```java
public class Outer {
    public foo(final int x) {
        final int y = 100;
        public class MethodInner {
            int z = 100;
            public int add() {
                return x + y + z;
            }
        }
    }
}
```

代码块中的内部类。  

```java
public class Outer {
    private int num;

    {
        final int x = 100;
        final int y = 100;
        class BlockInner {
            int z = 100;
            public int add() {
                return x + y + z;
            }
        }
        BlockInner blockInner = new BlockInner();
        num = blockInner.add();
    }
}
```

# 参考

> [关于对象与闭包的关系的一个有趣小故事 ](http://rednaxelafx.iteye.com/blog/245022)  

> [JVM 的规范中允许编程语言语义中创建闭包 (closure) 吗？](https://www.zhihu.com/question/27416568/answer/36565794)  

> [为什么 Java 闭包不能通过返回值之外的方式向外传递值？](https://www.zhihu.com/question/28190927/answer/39786939)