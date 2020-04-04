---
title: Java 8 之 Optional
date: 2020/3/19 20:38:0
tags: [Java]
categories: [Java]
---

在 Java 中，空指针异常（NullPointerException）是非常讨厌的。在 JDK 8 以前，我们通常只能通过主动防御式的检查来减少它的出现，但是在 JDK 8 之后我们可以使用 Optional 更优雅地处理这类问题。

<!--more-->

# 为缺失值建模
Java 一直试图让我们意识不到指针的存在，但是由于历史的原因，有一个例外就是 null 指针。其实 null 自身没有任何语义，它代表的是在静态类型语言中对缺失变量值的一种错误的建模方式。因为它的存在，我们不得不在可能出现空指针异常的地方添加检测的代码，这会得我们的代码膨胀，同时可读性下降。

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

将 Person 的 car 字段声明为 Optional，这可以非常清晰地表达一个人可能有车，也可能没车的情况，同理汽车的保险也是这样。但是保险一定会有保险公司，所以公司名称没有使用 Optional，这就非常清楚地表明了保险必须提供公司的名称。这样在解引用保险公司名称时，如果发生空指针异常，我们就可以非常确定地知道出错的原因，不再需要为其添加非空的检查，这种检查只会掩盖问题，并不能真正地修复问题。

> 需要强调的是，引入 Optional 的意图并不是要消除每一个 null 引用，与此相反，它的目标是帮助我们更好地设计出普适的 API，让编程人员在看到方法签名时就能了解到它是否接受一个 Optional 的值，这样会让我们更积极地将变量从 Optional 中解包出来，直面缺失的变量值。

虽然使用 Optional 作为类字段的类型是一个不错的做法，但是 Optional 类的设计者的初衷仅仅是要支持能返回 Optional 对象的语法，因此它并没有实现 Serializable 接口。如果我们在域模型中使用 Optional，同时又使用了某些要求序列化的库，那么很有可能会出现问题，折中的方案是提供一个能访问声明为 Optional、变量值可能缺失的接口，比如：

```java
public class Person {
    private Car car;
    public Optional<Car> getCarAsOptional() {
        return Optional.ofNullable(car);
    }
}
```

# 使用 Optional 的几种模式
我们可以使用 Optional 的 empty 静态工厂方法来创建一个空的 Optional 对象。

```java
Optional<Car> car = Optional.empty();
```

通过一个非空的值创建 Optional 可以使用 Optional 的 of 静态工厂方法，该方法传入的值不能为空，如果为空则会立刻抛出空指针异常，而不会等到我们试图访问时才抛出。

```java
Optional<Car> car = Optional.of(car);
```

如果想创建一个可以接受空值的 Optional 对象，那么可以使用 Optional 的 ofNullable 静态工厂方法，该方法会判断传入的值是否为空，如果为空则会创建一个空的 Optional 对象。

```java
Optional<Car> car = Optional.ofNullable(car);
```

## 使用 map
```java
// 以前的做法
String companyName = null;
if (insurance != null) {
    companyName = insurance.getCompanyName();
}

// 构建一个允许空值的 Optional
Optional<Insurance> optInsurance = Optional.ofNullable(insurance);
// 使用 map 提取和转换值
Optional<String> companyName = optInsurance.map(Insurance::getCompanyName);
```

## 使用 flatMap
```java
// 以前的做法
String companyName = null;
if (person != null) {
    Car car = person.getCar();
    if (car != null) {
        Insurance insurance = car.getInsurance();
        if (insurance != null) {
            companyName = insurance.getCompanyName();
        }
    }
}

// 使用 flatMap
Optional<Person> optPerson = Optional.ofNullable(person);
// 如果 person 为空，那么 flatMap 会返回一个空的 Optional 对象
String companyName = optPerson.flatMap(Person::getCar)
        .flatMap(Car::getInsurance)
        .map(Insurance::getCompanyName)
        .orElse("Unknown");
```

## 两个 Optional 的组合
假设我们有这样一个方法，它接受一个 Person 和一个 Car 对象，并以此为条件进行数据查询，最终返回一个满足该组合的最便宜的保险公司。

```java
public Insurance findCheapestInsurance(Person person, Car car) {
    // 数据查询业务逻辑
    return cheapestInsurance;
}
```

如果我们想提供一个该方法的“空”安全的版本，它接受两个 Optional 对象为参数，返回值为 Optional&lt;Insurance&gt; 对象，如果传入的任何对象为空，那么它的返回值也为空。我们可以通过 isPresent 方法来处理：

```java
public Optional<Insurance> nullSafeFindCheapestInsurance(Optional<Person> person, Optional<Car> car) {
    if (person.isPresent() && car.isPresent()) {
        return Optional.of(findCheapestInsurance(person.get(), car.get()));
    } else {
        return Optional.empty();
    }
}
```

这种方式虽然简单直观，但是和我们空检查的方式很像，这里有一种更好的实现方式：

```java
public Optional<Insurance> nullSafeFindCheapestInsurance(Optional<Person> person, Optional<Car> car) {
    // 对 person 调用 flatMap 方法，如果 person 的值为空则直接返回空的 Optional 对象
    // 如果不为空，那么会作为参数传递。对 car 调用 map 方法同理
    return person.flatMap(p -> car.map(c -> findCheapestInsurance(p, c)));
}
```

## 使用 filter
```java
if (insurance != null && "DiegoInsurance".equals(insurance.getCompanyName())) {
    System.out.println("do something...")
}

// 使用 filter
Optional<Insurance> optInsurance = ...;
optInsurance.filter(insurance -> "DiegoInsurance".equals(insurance.getCompanyName()))
        .ifPresent(insurance -> System.out.println("do something..."));
```

# 小结
方法 | 描述
---|---
map(Function&lt;? super T, ? extends U&gt; f) | 如果变量不为空则对该值执行传入的函数，否则返回一个空的 Optional 对象
flatMap(Function&lt;? super T, Optional&lt;U&gt;&gt; f) | 与 map 类似，但函数不同
of(T t) | 将指定值封装后返回，如果值为空则会直接抛出空指针异常
ofNullable(T t) | 将指定值封装后返回，如果值为空则会直接返回一个空的 Optional 对象
get() | 变量为空时抛出 NoSuchElementException 异常，在非常确定变量不为空时使用
orElse(T t) | 可以在变量为空时返回指定一个默认值
orElseGet(Supplier&lt;? extends T&gt; s) | 在变量为空时才会调用 Supplier 的 get 方法，可以在创建默认值比较耗时费力时使用
orElseThrow(Supplier&lt;? extends X&gt; e) | 与 get 方法类似，不过可以自定义抛出的异常
isPresent() | 判断变量是否为空
ifPresent(Consumer&lt;? super T&gt; c) | 可以在变量存在时执行作为参数传入的方法，否则不做任何操作

