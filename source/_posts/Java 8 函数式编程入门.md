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
public static List<Apple> filterApples(List<Apple> list, ApplePredicate applePredicate) {
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
filterApples(list, new ApplePredicate() {
    @Override
    public boolean test(Apple apple) {
        return apple.getWeight() > 150;
    }
});
```

从表面上看，我们并没有使用 class 创建类，但是实际上 Java 编译器会为匿名类生成一个 `ClassName$1` 这种形式的类文件。生成大量的类文件是不利的，因为每个类文件在使用时都需要加载和验证，这会影响应用的启动性能。在 Java 8 中，我们可以使用 Lambda 表达式来解决这个问题。

```java
filterApples(list, apple -> apple.getWeight() > 150);
```

# Lambda 表达式
我们可以把 Lambda 表达式理解为简洁地表示可传递的匿名函数的一种方式，它没有名称，但是它有参数列表、函数主体和返回类型，可能还有一个可以抛出的异常列表。

## 语法
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

## 在哪里使用
在函数式接口上使用 Lambda 表达式，而函数式接口就是**只定义了一个抽象方法的接口**。

一个典型的函数式接口就是 `java.lang.Runnable`，它只有一个抽象方法 `run()`，因此我们可以这样使用它：

```java
Runnable task = () -> System.out.println(Thread.currentThread());
Thread thread = new Thread(task, "thread-0");

// 或者直接传入 lambda 表达式
Thread thread = new Thread(() -> System.out.println(Thread.currentThread()), "thread-0");
```

## 函数描述符
Lambda 表达式有参数列表也有返回类型等，这些一起组成了 Lambda 表达式的签名。实际上函数式接口的抽象方法的签名基本上就是 Lambda 表达式的签名，我们将这个抽象方法叫做**函数描述符（Function Descriptor）**，并且我们使用特殊的表示法来描述 Lambda 表达式和函数描述符的签名。比如：`() -> void` 代表了参数列表为空，且返回 void 的函数。下面列举几个可以根据函数描述符判断 Lambda 表达式是否有效的例子。

```java
// 有效，因为 Runnable 的签名为 () -> void
public void execute(Runnable r) {
    r.run();
}
execute(() -> {});

// 有效，因为 fetch() 方法的签名为 () -> String
public static Callable<String> fetch() {
    return () -> "Hello World";
}

// 无效，因为 Predicate 接口 test 方法的签名为 (Apple) -> boolean
Predicate<Apple> p = (Apple a) -> a.getWeight();
```

## 环绕执行模式
在资源处理（比如处理文件或数据库）时，一个常见的模式就是打开一个资源，进行一些处理，最后关闭该资源。这就是所谓的环绕执行模式，在该模式中开始和结束部分总是很类似，只有中间执行处理的部分不同，因此中间这一部分就很适合进行行为参数化的操作。比如下面这部分代码：

```java
public String processFile() throws IOException {
    try (BufferedReader bufferedReader = new BufferedReader(new FileReader("data.txt"))) {
        return bufferedReader.readLine();
    }
}
```

try 包裹的资源能够在操作结束时隐式地关闭，此时的中间部分只有从文件中读取一行这一操作，我们将其行为参数化后，使整个方法扩展为能够根据传入参数的不同而执行不同的操作，比如：`String r = processFile((Buffered br) -> br.readLine() + br.readLine());`，很明显方法的签名为：(BufferedReader) -> String，因此我们需要创建一个函数式接口，然后修改 processFile 方法。

```java
@FunctionalInterface
public interface BufferedReaderProcessor {
    String process(BufferedReader br) throws IOException;
}
```

```java
public String processFile(BufferedReaderProcessor processor) throws IOException {
    try (BufferedReader bufferedReader = new BufferedReader(new FileReader("data.txt"))) {
        return processor.process(bufferedReader);
    }
}
```

## 函数式接口
在 JDK 1.8 中，很多函数式接口都带有 `@FunctionalInterface` 的注解，这代表该接口是一个函数式接口。我们在设计函数式接口的时候，最好带着该注解，因为它可以使编译器检查接口是否是函数式接口，从而提前发现错误。

除了很多常用的函数式接口，在 `java.util.function` 包下还引入了几个新的函数式接口，主要包括 `Predicate`、`Consumer`、`Function` 和 `Supplier` 这几类。

**其中 Predicate 可以理解为谓语、断言，我们知道谓词是对主语动作状态或特征的描述，指出做什么（do waht）、是什么（what is this）和怎么样（how）**。`java.util.function.Predicate<T>` 接口的 test 抽象方法接受一个泛型 T 对象并返回一个布尔类型的值，因此该接口方法的实现描述的应该是传入的 T 对象是否具备某些动作状态或特征。上面筛选苹果的例子也可以使用该接口进行修改：

```java
public static List<Apple> filterApples(List<Apple> list, Predicate<Apple> predicate) {
    List<Apple> result = new ArrayList<>();
    for (Apple apple : list) {
        if (predicate.test(apple)) {
            result.add(apple);
        }
    }
    return result;
}
```

`java.util.function.Consumer<T>` 接口定义了一个 accept 抽象方法，该方法接受一个泛型 T 对象，没有返回值，我们可以理解为该方法的实现是对传入的 T 对象进行消费的操作。下面列举一个简单的例子：

```java
public static <T> void forEach(List<T> list, Consumer<T> c) {
    for (T t : list) {
        c.accept(t);
    }
}

forEach(Arrays.asList(1, 2, 3, 4), e -> System.out.println(e));
```

`java.util.function.Function<T, R>` 接口定义了一个 apply 抽象方法，该方法接受一个泛型 T 对象，返回一个泛型 R 对象，我们可以理解为该方法的实现是将传入的 T 对象转化成 R 对象。下面列举一个例子：

```java
public static <T, R> List<R> map(List<T> list, Function<T, R> f) {
    List<R> result = new ArrayList<>();
    for (T t : list) {
        result.add(f.apply(t));
    }
    return result;
}

map(Arrays.asList("Hello", "World"), (String s) -> s.length());
```

我们知道，在 Java 中泛型只能绑定到引用类型上，因此 Java 提供了自动拆箱和装箱的操作。但是这种操作需要付出性能代价，为装箱后的值本质上就是把原始类型包裹起来并保存到堆上，装箱后的值需要更多的内存，并需要额外的内存搜索来获取被包裹的原始值。为了避免在使用这些函数式接口时出现自动装箱的操作，JDK 8 专门为这些接口提供了使用原始类型的版本。比如 IntPredicate、IntConsumer、LongToIntFunction 等。下面附上一些总结的使用案例：

使用案例 | Lambda 的例子 | 对应的函数式接口
---|---|---
布尔表达式 | (List&lt;String&gt; list) -> list.isEmpty() | Predicate&lt;List&lt;String&gt;&gt;
消费一个对象 | (Apple a) -> System.out.println(a.getWeight()) | Consumer&lt;Apple&gt;
从一个对象中选择或提取 | (String s) -> s.length() | Function&lt;String, Integer&gt; 或 ToIntFunction&lt;String&gt;
合并两个值 | (int a, int b) -> a * b | IntBinaryOperrator
比较两个对象 | (Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight()) | Comparator&lt;Apple&gt; 或 BiFunction&lt;Apple, Apple, Integer&gt; 或 ToIntBiFunction&lt;Apple, Apple&gt;