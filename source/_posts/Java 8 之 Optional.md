---
title: Java 8 之 Optional
date: 2020/3/19 20:38:0
tags: [Java]
categories: [Java]
---

在 Java 中，空指针异常（NullPointerException）是非常讨厌的。在 JDK 8 以前，我们通常只能通过主动防御式的检查来减少它的出现，但是在 JDK 8 之后我们可以使用 Optional 更优雅地处理这类问题。

<!--more-->

Java 一直试图让我们意识不到指针的存在，但是由于历史的原因，有一个例外就是 null 指针。其实 null 自身没有任何语义，它代表的是在静态类型语言中对缺失变量值的一种错误的建模方式。因为它的存在，我们不得不在可能出现空指针异常的地方添加检测的代码，这会使得我们的代码膨胀，同时可读性下降。

近几年出现的很多语言都尝试去解决这个问题，比如 Groovy 通过引入安全导航操作符使得我们可以安全地访问可能为 null 的变量，但是这种处理方式并不值得被提倡，原因是在遇到变量为 null 时我们应该结合当前的算法和数据模型来考虑是否需要先返回一个 null，如果一碰到空指针就加 if 条件判断，那么其实并没有真正的解决问题，这只是暂时的掩盖了问题。

有一些函数式编程语言，比如 Haskell、Scala 等则试图从另一个角度来解决这个问题。比如在 Haskell 中有一个 Maybe 类型，它的变量可以是任意类型，也可以什么都不是。Java 8 就是从 Haskell 和 Scala 中汲取的灵感，引入了 `java.util.Optional<T>` 类。当变量存在时，Optional 只是对类进行简单封装；当变量不存在时，缺失的值会被建模成一个“空”的 Optional 对象，由 Optional.empty() 方法返回。从语义上来讲，我们可以把 null 和 Optional.empty() 当作一回事，而使用 Optional 建模的优势就是：如果我们尝试解引用一个 null 则会抛出空指针异常，而 Optional.empty() 则不会，因为它本身就是一个 Optional 类的有效对象。与此同时，使用 Optional 还可以丰富我们模型的语义，当系统出现问题时更容易排查，比如：

```java
public class Person {
    private Optional<Car> car;
    public Optional<Car> getCar() {
        return car;
    }
}

public class Car {
    private Optional<Insurance> insurance;
    public Optional<Insurance> getInsurance() {
        return insurance;
    }
}

public class Insurance {
    private String companyName;
    public String getCompanyName() {
        return companyName;
    }
}
```

将 Person 的 car 字段声明为 Optional，这可以非常清晰地表达一个人可能有车，也可能没车的情况，同理汽车的保险也是这样。但是保险一定会有保险公司，所以公司名称没有使用 Optional，这就非常清楚地表明了保险必须提供公司的名称。这样在解引用保险公司名称时，如果发生空指针异常，我们就可以非常确定地知道出错的原因，不再需要为其添加非空的检查，这种检查只会掩盖问题，并不能真正地修复问题。**需要强调的是，引入 Optional 的意图并不是要消除每一个 null 引用，与此相反，它的目标是帮助我们更好地设计出普适的 API，让编程人员在看到方法签名时就能了解到它是否接受一个 Optional 的值，这样会让我们更积极地将变量从 Optional 中解包出来，直面缺失的变量值。**