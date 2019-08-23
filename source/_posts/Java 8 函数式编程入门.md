---
title: Java 8 函数式编程入门
date: 2019/7/25 16:24:0
tags: [Java]
categories: [Java]
---

Java 8 发布至今已经有好长一段时间了，但是在实际工作中使用函数式编程的机会还是太少，对于 Java 的函数式编程了解的还是不够深入，因此借着阅读《Java 8 in Action》的机会将自己的心得体会记录一下。

<!--more-->

我们知道，在 Java 中值的形式有两种：原始值和引用值。原始值就是那些基本类型的值，包括 int 类型的值，double 类型的值等。引用值就是那些引用类型的值，也就是对象的值。这些值能够在程序执行期间作为参数进行传递，因此又被称为**一等值（或者一等公民）**。与此同时，Java 中的类和方法由于无法作为参数传递而被称为**二等公民**。但是很多编程语言的实践证明了让方法作为一等值可以使编程变得更加容易，因此 Java 的设计者们将这个功能加入到了 JDK 8 中，从而使方法可以作为值进行传递。

# 行为参数化
行为参数化简单来说就是将一个代码块准备好却不马上执行，这部分代码可以作为参数传递给另一个方法，这意味着我们可以推迟这部分代码的执行。行为参数化是处理频繁的需求变更的一种良好的开发模式。下面使用书上的例子进行详细说明。

## 用例子引出行为参数化
给定一个苹果集合，筛选出绿颜色的苹果。

```java
public static List<Apple> filterGreenApples(List<Apple> list) {
    List<Apple> result = new ArrayList<>();
    for (Apple apple : list) {
        if ("green".equals(apple.getColor())) {
            result.add(apple);
        }
    }
    return result;
}
```

这样是可以筛选出绿色的苹果，但是我们可以更进一步，写一个可以筛选任意颜色苹果的方法。

```java
public static List<Apple> filterGreenApples(List<Apple> list, String color) {
    List<Apple> result = new ArrayList<>();
    for (Apple apple : list) {
        if (apple.getColor().equals(color)) {
            result.add(apple);
        }
    }
    return result;
}
```

正在我们沾沾自喜时，需求又变了，要我们筛选出颜色为绿色，同时重量超过 150g 的苹果。

```java
public static List<Apple> filterGreenApples(List<Apple> list, String color, int weight) {
    List<Apple> result = new ArrayList<>();
    for (Apple apple : list) {
        if (apple.getColor().equals(color) && apple.getWeight() > weight) {
            result.add(apple);
        }
    }
    return result;
}
```

其实到这里，我们差不多已经能够发现这种写法并不能很好地应对需求变更，假如又加入了产地、品牌、形状等筛选条件，我们还需要重新修改方法签名和实现，不够灵活也不方便维护。此时我们可以试着从更高层级的抽象入手，一种可能的方案是对我们选择的标准建模：我们需要根据苹果的某些属性值来返回一个 boolean 值，我们可以把它抽象成一个返回 boolean 值的函数，这个函数很像我们语法中主谓结构的谓词部分。

```java
public interface ApplePredicate {
    boolean test(Apple apple);
}
```

现在可以根据不同的选择标准进行不同的实现了。

```java
/**
* 超过 150g 的苹果
*/
public class AppleHeavyWeightPredicate implements ApplePredicate {
    @Override
    public boolean test(Apple apple) {
        return apple.getWeight() > 150;
    }
}
```

```java
/**
* 绿颜色的苹果
*/
public class AppleGreenColorPredicate implements ApplePredicate {
    @Override
    public boolean test(Apple apple) {
        return "green".equals(apple.getColor());
    }
}
```

此时还需要修改一下筛选的方法。

```java
public static List<Apple> filterGreenApples(List<Apple> list, ApplePredicate applePredicate) {
    List<Apple> result = new ArrayList<>();
    for (Apple apple : list) {
        if (applePredicate.test(apple)) {
            result.add(apple);
        }
    }
    return result;
}
```

接下来如果需求变更，我们只需要新建一个谓词实现类即可。但是我们很快就会发现新建实现类是很麻烦的，一个很容易想到的方式就是匿名类。我们可以在调用筛选方法时传入一个匿名类。

```java
filterGreenApples(list, new ApplePredicate() {
    @Override
    public boolean test(Apple apple) {
        return apple.getWeight() > 150;
    }
});
```

从表面上看，我们并没有使用 class 创建类，但是实际上 Java 编译器会为匿名类生成一个 `ClassName$1` 这种形式的类文件。生成大量的类文件是不利的，因为每个类文件在使用时都需要加载和验证，这会影响应用的启动性能。在 Java 8 中，我们可以使用 Lambda 表达式来解决这个问题。

```java
filterGreenApples(list, apple -> apple.getWeight() > 150);
```

# Lambda 表达式
我们可以把 Lambda 表达式理解为简洁地表示可传递的匿名函数的一种方式，它没有名称，但是它有参数列表、函数主体和返回类型，可能还有一个可以抛出的异常列表。

## Lambda 表达式语法
Lambda 表达式有三部分组成，参数列表、箭头和 Lambda 主体。基本语法为：

```
// 参数列表、箭头和表达式（注意此处的表达式不带分号）
(parameters) -> expression

// 参数列表、箭头和语句（这里的语句需要用花括号包含，语句需要分号）
(parameters) -> { statements; }
```

下面列举几个 Lambda 表达式的正例和反例。

```java
// 它是有效的，没有参数列表，返回值类型为 void，主体为空
() -> {}

// 它是有效的，没有参数列表，返回 String 作为表达式
() -> "Hello World!"

// 它是有效的，没有参数列表，使用显式的返回语句返回 String
() -> { return "Hello World!"; }

// 它是无效的，因为 return 是控制流语句，要使此表达式有效，语句需要用花括号包含
(Integer i) -> return "Hello World!" + i;

// 它是无效的，因为 "Hello World!" 是一个表达式，不是语句，所以应该把花括号和分号去掉
(String s) -> { "Hello World!"; }
```

## 在哪里使用 Lambda 表达式
在函数式接口上使用 Lambda 表达式，而函数式接口就是**只定义了一个抽象方法的接口**。

一个典型的函数式接口就是 `java.lang.Runnable`，它只有一个抽象方法 `run()`，因此我们可以这样使用它：

```java
Runnable task = () -> System.out.println(Thread.currentThread());
Thread thread = new Thread(task, "thread-0");

// 或者直接传入 lambda 表达式
Thread thread = new Thread(() -> System.out.println(Thread.currentThread()), "thread-0");
```

## 函数描述符（Function Descriptor）
Lambda 表达式有参数列表也有返回类型等，这些一起组成了 Lambda 表达式的签名。实际上函数式接口的抽象方法的签名基本上就是 Lambda 表达式的签名，我们将这个抽象方法叫做**函数描述符**，并且我们使用特殊的表示法来描述 Lambda 表达式和函数描述符的签名。比如：`() -> void` 代表了参数列表为空，且返回 void 的函数。

## 函数式接口
在 JDK 1.8 中，很多函数式接口都带有 `@FunctionalInterface` 的注解，这代表该接口是一个函数式接口。我们在设计函数式接口的时候，最好带着该注解，因为它可以使编译器检查接口是否是函数式接口，从而提前发现错误。

除了很多常用的函数式接口，在 `java.util.function` 包下还引入了几个新的函数式接口，主要包括 `Predicate`、`Consumer`、`Function` 和 `Supplier` 这几类。

