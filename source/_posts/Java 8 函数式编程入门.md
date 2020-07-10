---
title: Java 8 函数式编程入门
date: 2019/7/25 16:24:0
tags: [Java]
categories: [Java]
---

Java 8 发布至今已经有好长一段时间了，但是在实际工作中使用函数式编程的机会还是太少，对于 Java 的函数式编程了解的还是不够深入，因此借着阅读《Java 8 in Action》的机会将自己的心得体会记录一下。

<!--more-->

我们知道，在 Java 语言层面的值的形式有两种：原始值和引用值。原始值就是那些基本类型的值，包括 int 类型的值，double 类型的值等。引用值就是那些引用类型的值，也就是对象的地址值。这些值能够在程序执行期间作为参数进行传递，因此又被称为**一等值（或者一等公民）**。与此同时，Java 中的类和方法由于无法作为参数传递而被称为**二等公民**。但是很多编程语言的实践证明了让方法作为一等值可以使编程变得更加容易，因此 Java 的设计者们将这个功能加入到了 JDK 8 中，从而使方法可以作为值进行传递。

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

## 方法引用
方法引用使得我们可以重复使用现有的方法定义，并像 Lambda 一样传递它们。当我们使用方法引用时，我们需要将目标引用放在分隔符 `::` 前面，方法名称放在后面，比如 `Apple::getWeight` 就是一个方法引用。它的基本思想是，如果一个 Lambda 表达式代表的只是“直接调用这个方法”，那最好还是用名称来调用它，而不是去描述如何调用它。下面列举一些例子来说明：

Lambda | 等效的方法引用
---|---
(Apple a) -> a.getWeight() | Apple::getWeight
() -> Thread.currentThread().dumpStack() |  Thread.currentThread::dumpStack
(str, i) -> str.substring(i) | String::substring
(String s) -> System.out.println(s) | System.out::println

方法引用主要有三种，一种是指向静态方法的方法引用，比如 Integer 的 parseInt 方法，对应的方法引用为 Integer::parseInt。一种是指向任意类型实例方法的方法引用，这一类方法引用的特点就是当我们在引用一个对象的方法时，这个对象本身又是 Lambda 中的一个参数，比如 (String s) -> s.toUpperCase() 对应的方法引用为 String::toUpperCase。还有一种是指向现有对象的实例方法的方法引用，这一类方法引用的特点是在 Lambda 中调用一个在外部环境已经存在的对象中的方法。

![方法引用](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003231726/2020/03/23/2jR.png)

## 构造函数引用
对于一个现有的构造函数，我们可以利用它的名称和关键字 new 来创建一个它的引用：ClassName::new。假如有一个构造函数没有参数，那么它就与 `Supplier<T>` 接口的方法签名 `() -> T` 一致，所以我们可以这样做：

```java
Supplier<Apple> c1 = Apple::new; // 构造函数引用指向默认的 Apple() 构造函数
Apple a1 = c1.get(); // 调用 Supplier 的 get 方法才会真正创建一个 Apple 对象

// 这就等价于

Supplier<Apple> c2 = () -> new Apple();
Apple a2 = c2.get();
```

如果构造函数的签名是 Apple(Integer weight)，那么它就与 Function 接口的签名 `(T, R) -> R` 一致，所以可以这样做：

```java
Function<Integer, Apple> f1 = Apple::new; // 指向 Apple(Integer weight) 的构造函数引用
Apple a1 = f1.apply(120); // 调用该 Function 接口的 apply 方法，并给出要求的重量，产生一个新的对象

// 等价于

Function<Integer, Apple> f2 = (Integer weight) -> new Apple(weight);
Apple a2 = f2.apply(120);
```

如果构造函数的签名为 Apple(String color, Integer weight)，那么就与 BiFunction 接口的签名 `(T, U, R) -> R` 一致。

```java
BiFunction<String, Integer, Apple> f1 = Apple::new;
Apple a1 = f1.apply("red", 120);

// 等价于
BiFunction<String, Integer, Apple> f2 = (String color, Integer weight) -> new Apple(color, weight);
Apple a2 = f2.apply("red", 120);
```

## 引用总结
- 静态方法引用：`ClassName::methodName`
- 实例上的实例方法引用：`instanceReference::methodName`
- 超类上的实例方法引用：`super::methodName`
- 类型上的实例方法引用：`ClassName::methodName`
- 构造方法引用：`Class::new`
- 数组构造方法引用：`TypeName[]::new`

## 复合方法
很多函数式接口都提供了进行复合的方法（以默认方法的方式提供），比如用于传递 Lambda 表达式的 Comparator、Function 和 Predicate 接口。允许使用复合的方法意味着我们可以将多个简单的 Lambda 表达式复合成较为复杂的表达式，从而实现更加复杂的需求，比如我们可以让两个谓词进行 or 操作，从而组合成一个更大的谓词。

### 比较器复合
```java
// 使用 Comparator 的静态方法 comparing，根据提取用于比较的键值的 Function 来返回一个 Comparator
Comparator<Apple> c = Comparator.comparing(Apple::getWeight);

// 逆序
Comparator<Apple> c1 = Comparator.comparing(Apple::getWeight).reversed();

// 比较器链：如果两个苹果重量相同，使用重量无法比较出区别，那么可以继续使用其他的比较器进行比较
Comparator<Apple> c2 = Comparator.comparing(Apple::getWeight)
                .reversed()
                .thenComparing(Apple::getColor);

```

### 谓词复合
and 和 or 方法是按照在表达式链中的位置从左到右确定优先级的，比如 `a.or(b).and(c)` 可以看作 `(a || b) && c`。

```java
// 谓语：苹果是红色的
Predicate<Apple> p1 = (Apple a) -> "red".equals(a.getColor());

// 非：苹果不是红色的
Predicate<Apple> p2 = p1.negate();

// 与：苹果既是红色的又大于 110 克
Predicate<Apple> p3 = p1.and((Apple a) -> a.getWeight() > 110);

// 或：苹果可能是大于 110 克的红苹果，也可能是绿苹果
Predicate<Apple> p4 = p3.or((Apple a) -> "green".equals(a.getColor()));
```

### 函数复合
andThen 方法会返回一个 Function，它先对输入应用一个函数，再对输出应用另一个函数。比如，有个函数 f 是给数字加 1，另一个函数是给数字乘 2，我们可以将这两个函数组合起来，先加 1 再乘 2。

```java
Function<Integer, Integer> f = x -> x + 1;
Function<Integer, Integer> g = x -> x * 2;
Function<Integer, Integer> h = f.andThen(g); // 数学上会写作 g(f(x))
int result = h.apply(1); // 结果为 4
```

如果在上面的例子中使用 compose 方法，那就意味着结果为 `f(g(x))`。

```java
Function<Integer, Integer> f = x -> x + 1;
Function<Integer, Integer> g = x -> x * 2;
Function<Integer, Integer> h = f.compose(g); // 数学上会写作 f(g(x))
int result = h.apply(1); // 结果为 3
```

# 流
Stream 不是集合元素，它不是数据结构并不保存数据，它是有关算法和计算的，它更像一个高级版本的 Iterator。用户在使用普通的 Iterator 时，只能一个一个显式地遍历元素并对其执行某些操作（外部迭代）；而在使用 Stream 时，用户只需给出对其包含的元素执行什么样的操作即可，比如“过滤出长度大于 10 的字符串”、“获取每个字符串的首字母”等，流会隐式地在内部进行遍历（内部迭代），并做出相应的数据转换。

与迭代器类似，Stream 是单向的，不可往复，即数据只能遍历一次，遍历过一次后就用尽了，就像流水从面前流过，一去不复返。与迭代器不同的是，Stream 可以并行化操作，而迭代器只能命令式地、串行化地操作。当使用串行方式去遍历时，每个 item 读完后再读下一个 item。而使用并行去遍历时，数据会被分成多个段，其中每一段都在不同的线程中处理，最终将结果合并。Stream 的并行操作依赖于 Java 7 中引入的 `Fork/Join` 框架（[JSR 166y](http://gee.cs.oswego.edu/cgi-bin/viewcvs.cgi/jsr166/src/jsr166y/)）来拆分任务和加速处理过程。

## 为什么要使用流
Stream 作为 Java 8 的一大亮点，它与 java.io 包里的 InputStream 和 OutputStream 是完全不同的概念。它也不同于 StAX 对 XML 解析的 Stream，也不是 Amazon Kinesis 对大数据实时处理的 Stream。Java 8 中的 Stream 是对集合对象功能的增强，它专注于对集合对象进行各种非常便利、高效的聚合操作（aggregate operation），或者大批量数据操作 (bulk data operation)。Stream API 借助于 Lambda 表达式，极大的提高编程效率和程序可读性。同时它提供串行和并行两种模式进行汇聚操作，使用并发模式能够充分利用多核处理器的优势。通常编写并行代码很难而且容易出错, 但使用 Stream API 无需编写一行多线程的代码就可以很方便地写出高性能的并发程序。所以说，Java 8 中首次出现的 Stream 是一个函数式语言 + 多核时代综合影响的产物。

在传统的 J2EE 应用中，Java 代码经常不得不依赖于关系型数据库的聚合操作来完成诸如：客户每月平均消费金额、最贵的在售商品、本周完成的有效订单、取十个数据样本作为首页推荐等等这类的操作，但在当今这个数据大爆炸的时代，数据的来源更加多样化，很多时候不得不脱离 RDBMS，或者以底层返回的数据为基础进行更上层的数据统计。而 Java 的集合 API 中，仅仅有极少量的辅助型方法，很多时候程序员需要用 Iterator 遍历集合并完成相关的聚合应用逻辑。

## 构建流
可以通过集合、值序列、数组、文件或者函数（类似于 Python 中的生成器）等来创建流。在 Java 8 中，Collection 接口被扩展，增加了两个默认方法来获取 stream。

```java
// 由集合创建
List<Apple> apples = new ArrayList<>();
Stream<Apple> stream = apples.stream();
```

```java
// 由值序列创建流
Stream<String> stream = Stream.of("Hello", "World");
```

```java
// 由数组创建流
int[] numbers = {2, 3, 5, 7, 9, 11};
IntStream stream = Arrays.stream(numbers);
```

java.nio.file.Files 中有很多静态方法都会返回一个流。比如 Files.lines 方法会返回一个指定文件中的各行构成的字符串流。

```java
// 由文件创建流
long uniqueWords = 0;
try (Stream<String> lines = Files.lines(Paths.get("data.txt"), Charset.defaultCharset())) {
    uniqueWords = lines.flatMap(line -> Arrays.stream(line.split(" "))) // 生成单词流
            .distinct() // 删除重复项
            .count();
}
```

Stream API 提供了两个静态方法来从函数生成流，包括 Stream.iterate() 和 Stream.generate()，由于这两个操作产生的流都会用给定的函数按需创建值，因此都可以创造出所谓的无限流。

```java
// 接受一个初始值 0，流的第一个元素为 0，然后为生成的新值 2，以此类推
Stream.iterate(0, n -> n + 2).limit(10).forEach(System.out::println);

// 斐波那契元组序列
Stream.iterate(new int[]{0, 1}, t -> new int[]{t[1], t[0] + t[1]})
        .limit(20)
        .forEach(t -> System.out.println("(" + t[0] + "," + t[1] + ")"));
```

与 iterate 不同，generate 不是依次对每个新生成的值应用函数的，它接受一个 `Supplier<T>` 类型的参数来提供新的值。

```java
Stream.generate(Math::random).limit(5).forEach(System.out::println);
```

## 流操作
流操作分为两种：中间操作和终端操作。中间操作包括 filter、map、sorted、limit、distinct 等，这类操作可以连接起来形成一个查询的操作链，并且因为中间操作一般都可以合并起来，所以它们都是惰性化的，只有在遇到终端操作时才会一次性全部处理。终端操作包括 forEach、collect、reduce、count 等，这类操作会执行中间操作链并产生结果。一个流只能有一个终端操作，当这个操作执行后，流就被用“光”了。

### 筛选和切片
Stream 的筛选主要通过 filter 方法实现，该方法接收一个谓词作为参数，并返回一个包含所有符合谓词的元素的流。当然还有一个 distinct 方法能够返回一个元素各异的流（根据流所生成元素的 hashCode 和 equals 方法实现），这个方法的作用与 SQL 中的 SELECT DISTINCT 语句类似。举个例子，下面的代码会筛选出列表中所有的偶数，并确保没有重复。

```java
List<Integer> numbers = Arrays.asList(1, 2, 1, 3, 3, 2, 4, 6);
numbers.stream()
        .filter(n -> n % 2 == 0)
        .distinct()
        .forEach(System.out::println);
```

Stream 的切片主要通过 limit 方法和 skip 方法来实现。limit(n) 方法会返回一个不超过给定长度的流，而 skip(n) 方法会返回一个扔掉了前 n 个元素的流，如果流中元素不足 n 个，则会返回一个空流。

### 映射
流的 map 方法接受一个函数（`Function<? super T, ? extends R> mapper`）作为参数，这个函数会被应用到每个元素上，并将其映射成一个新的元素。

```java
List<Integer> list = apples.stream()
        .map(Apple::getWeight)
        .collect(Collectors.toList());
```

除了 map 方法，Stream 还有一个将流扁平化的 flatMap 方法，该方法同样接受一个函数，但是这个函数与 map 方法接受的函数不同，它的声明为 `Function<? super T, ? extends Stream<? extends R>> mapper`，该函数会将流中的每个元素转换为另一个流。flatMap 方法会将流中每个元素都转换为另一个流，然后把所有的流连接起来成为一个新的流。比较 map 方法和 flatMap 方法我们会发现，map 适合一对一映射的场景，而 flatMap 适合一对多映射的场景。flatMap 方法的入参为多个列表，结果可以返回一个列表；而 map 方法如果接受多个列表，那么返回的结果也是多个列表。

### 查找和匹配
很多时候我们需要查看数据集中的某些元素是否匹配一个给定的属性，Stream API 就提供了类似的工具，包括 allMatch、anyMatch、noneMatch、findFirst 和 findAny，这些操作都用到了短路，类似于 Java 中 `&&` 和 `||` 运算符的短路。

> 有些操作不需要处理整个流就能得到结果。例如，假设你需要对一个用 and 连起来的大布尔表达式求值。不管表达式有多长，你只需找到一个表达式为 false，就可以推断整个表达式将返回 false，所以用不着计算整个表达式。这就是短路。对于流而言，某些操作不用处理整个流就能得到结果。只要找到一个元素，就可以有结果了。limit 就是一个短路操作：它只需创建一个给定大小的流。在碰到无限大小的流的时候，这种操作就有用了：它们可以把无限流变成有限流。

### 规约
规约可以将流中所有的元素反复结合最终得到一个值，比如“计算所有苹果的重量”、“所有苹果中最重的是哪个”等。

```java
// 正常外部迭代求和
int sum = 0;
for (int x : numbers) {
    sum += x;
}

// 使用规约求和
int sum1 = numbers.stream().reduce(0, (a, b) -> a + b);
// 利用 Integer 类的静态方法 sum
int sum2 = numbers.stream().reduce(0, Integer::sum);

// 没有初始值的求和，在没有初始值时，需要考虑流中没有任何元素的情况，因此返回值为 Optional 类型
Optional<Integer> sum3 = numbers.stream().reduce((a, b) -> (a + b));

// 求乘积
int product = numbers.stream().reduce(1, (a, b) -> a * b);

// 求最小值
Optional<Integer> min = numbers.stream().reduce(Integer::min);

// 求最大值
Optional<Integer> max = numbers.stream().reduce(Integer::max);

// 利用 map 和 reduce 求总数
int count = apples.stream()
        .map(a -> 1)
        .reduce(Integer::sum);

// 直接使用 count 方法求总数
long count1 = apples.stream().count();
```

对于 reduce 方法，如果未定义初始值，那么第一次执行时第一个参数的值就是流的第一个元素，第二个参数就是流的第二个元素；如果定义了初始值，则第一次执行时第一个参数的值就是初始值，第二个参数就是流的第一个元素。下面需要说明一个特殊的 reduce 方法。

```java
<U> U reduce(U identity,
                 BiFunction<U, ? super T, U> accumulator,
                 BinaryOperator<U> combiner);
```

第一个参数为实例 identity，表示要返回的 U 类型对象的初始化实例，第二个参数为累加器 accumulator，可以使用二元表达式（即二元 Lambda 表达式），声明在 identity 的基础上连续使用的逻辑，第三个参数为组合器 combiner，由于流是支持并发操作的，为了避免竞争，reduce 线程都会有独立的 result，combiner 的作用就是合并每个线程的 result 得到最终结果。这也说明了了第三个函数参数的数据类型必须为方法返回值的类型。

### 小结
方法 | 类型 | 函数描述符 | 描述
---|---|---|---
filter(Predicate&lt;T&gt; p) | 中间 | T -> boolean | 根据谓词筛选
distinct() | 中间（有状态-无界） | | 返回一个元素各异的流，即去重
limit(long n) | 中间（有状态-有界） | | 返回一个不超过给定长度的流
skip(long n) | 中间（有状态-有界） | | 返回一个扔掉了前 n 个元素的流
map(Function&lt;T, R&gt; f) | 中间 | T -> R | 根据函数将流中的每个元素映射为新的元素
flatMap(Function&lt;T, Stream&lt;R&gt;&gt; f) | 中间 | T -> Stream&lt;R&gt; | 将流中元素都转成新流并最终合并为一个流
sorted() | 中间（有状态-无界） | | 产生一个新流，其中按字典顺序排序
sorted(Comparator&lt;T&gt; c) | 中间（有状态-无界） | (T, T) -> int | 产生一个新流，其中按比较器排序
allMatch(Predicate&lt;T&gt; p) | 终端 | T -> boolean | 检查是否匹配所有元素
anyMatch(Predicate&lt;T&gt; p) | 终端 | T -> boolean | 检查是否至少匹配一个元素
noneMatch(Predicate&lt;T&gt; p) | 终端 | T -> boolean | 检查是否没有匹配所有元素
findFirst() | 终端 | | 返回第一个元素
findAny() | 终端 | | 返回当前流中的任意元素
forEach(Consumer&lt;T&gt; c) | 终端 | T -> void | 内部迭代
reduce(BinaryOperator&lt;T&gt; b) | 终端（有状态-有界） | (T, T) -> T | 将流中所有的元素反复结合最终得到一个值
count() | 终端（有状态-有界） | | 计算流中元素的个数（**规约操作**）
min(Comparator&lt;T&gt; c) | 终端（有状态-有界） | (T, T) -> int | 获取流中最小的元素（**规约操作**）
max(Comparator&lt;T&gt; c) | 终端（有状态-有界） | (T, T) -> int | 获取流中最大的元素（**规约操作**）
collect(Collector&lt;T, A, R&gt; c) | 终端 | | 接受各种做法将流中元素汇总成一个（**规约操作**）

## 收集器
流的 collect 方法其实也是一个归约操作，就像 reduce 一样可以接受各种做法作为参数，将流中的元素累积成一个汇总结果，具体的做法可以使用预定义的 Collector 接口的实现，也就是 Collectors 类提供的一系列的静态方法（工厂方法），这些方法主要提供了三类功能：将流中元素规约汇总为一个值，元素分组以及元素分区。

### 规约与汇总

```java
// 查找最大值和最小值
Comparator<Apple> comparator = Comparator.comparingInt(Apple::getWeight);
Optional<Apple> max = apples.stream().collect(Collectors.maxBy(comparator));

// 求总数
long count = apples.stream().collect(Collectors.counting());

// 求和
int sum = apples.stream().collect(Collectors.summingInt(Apple::getWeight));

// 求平均值
double avg = apples.stream().collect(Collectors.averagingInt(Apple::getWeight));

// 连接字符串
// joining 方法返回的收集器会把流中每个元素应用 toString 方法得到的所有字符串连成一个
String colors = apples.stream().collect(Collectors.joining());

String colors1 = apples.stream().map(Apple::getColor).collect(Collectors.joining(", "));
```

事实上，很多收集器都是可以用 reducing 工厂方法定义的规约过程的特殊情况而已，特化的目的是为了方便编程人员。reducing 方法有两种，一种是单参数方法，另一种是三参数方法。从逻辑上说，reducing 的原理是利用累积函数，把一个初始化为起始值的累加器，和把转换函数应用到流中每个元素上得到的结果不断迭代合并。

```java
int total = apples.stream()
        .collect(Collectors.reducing(0, // 初始值
                Apple::getWeight, // 转换函数
                Integer::sum)); // 累积函数
```

我们可以将单参数的 reducing 方法看作三参数方法的特殊情况，它把流中第一个元素作为起点，把恒等函数（即一个函数仅仅是返回其输入参数）作为一个转换函数。

```java
// 求最大值
Optional<Apple> max = apples.stream()
        .collect(Collectors.reducing((a1, a2) -> a1.getWeight() > a2.getWeight() ? a1 : a2));
```

### 分组
使用 groupingBy 时需要提供一个分类函数，通过它将流中的元素划分到不同的组中。

```java
// 按照颜色分组
Map<String, List<Apple>> groups = apples.stream().collect(Collectors.groupingBy(Apple::getColor));

// 编写分组函数
Map<String, List<Apple>> groups = apples.stream().collect(Collectors.groupingBy(apple -> {
    if (apple.getWeight() < 120) {
        return "LIGHT"; // 轻的
    } else {
        return "HEAVY"; // 重的
    }
}));
```

多级分组可以使用双参数版本的 groupingBy 方法，它除了接受一个分类函数外，还可以接受一个 Collector 类型的参数。

```java
Map<String, Map<String, List<Apple>>> groups = apples.stream()
        .collect(Collectors.groupingBy(Apple::getColor, // 一级分组
                Collectors.groupingBy(apple -> { // 二级分组
                    if (apple.getWeight() < 120) {
                        return "LIGHT";
                    } else {
                        return "HEAVY";
                    }
                })));
```

多级分组可以由两级扩展到任意层级。一般把 groupingBy 看作“桶”比较容易理解，第一个 groupingBy 给每个键建立了一个桶，然后再用下游的收集器去收集每个桶中的元素，以此得到 n 级分组。进一步的，传递给第一个 groupingBy 的第二个收集器可以是任何类型。实际上单参数的 groupingBy(f) 只是 groupingBy(f, Collectors.toList()) 的简便写法。

```java
// 返回的 Map 类似：{"green": 3, "red": 5}
Map<String, Long> groups = apples.stream()
                .collect(Collectors.groupingBy(Apple::getColor, Collectors.counting()));
```

### 分区
分区是分组的特殊情况，因为在分区中分类函数是一个谓词，这意味着分组 Map 的键是 boolean 类型的，它最多可以分为两组：true 是一组，false 是另一组。

```java
Map<Boolean, List<User>> partition = users.stream().collect(Collectors.partitioningBy(User::isVip);
```
