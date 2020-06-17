---
title: Dubbo SPI 机制
date: 2020/6/17 9:54:0
tags: [Dubbo]
categories: [分布式]
---

Dubbo 良好的扩展性得益于它的 SPI 机制，在 Dubbo 中，几乎所有的功能组件都是基于 SPI 机制实现的。由于 Java SPI 机制存在一些问题，且无法满足 Dubbo 的需求，于是 Dubbo SPI 就在 Java SPI 的思想上做了改进，形成了一套自己的配置规范和特性。

<!--more-->

# 改进
Java SPI 在加载插件的时候，会一次性地**加载并实例化**扩展点所有的实现类，因此这个过程可能很耗时，同时可能有的类并没有用到，这也会造成浪费。而 Dubbo SPI 在加载插件的时候会缓存加载过的类和实例化后的对象，同时缓存的 Class 并不会全部实例化，而是按需实例化并缓存，因此性能更好。

当 Java SPI 加载插件失败时，可能会因为各种原因导致异常信息被“吞掉”，而 Dubbo SPI 在扩展加载失败的时候会先抛出真实的异常并打印日志。扩展点在被动加载的时候，即使有部分扩展加载失败也不会影响其他扩展点和整个框架的使用。

Dubbo SPI 还增加了对扩展的 IoC 和 AOP 的支持，一个扩展可以通过 setter 直接注入其他扩展。同时 Dubbo 还支持包装扩展类，它推荐把通用的抽象逻辑放到包装类中，用于实现扩展点的 AOP 特性。比如 ProtocolFilterWrapper 类就是一个包装类，它包装了一个 Protocol，将一些通用的判断逻辑全部放在了 export 方法中，但最终它还是会调用 Protocol#export 方法。这类似于代理模式，在被代理的类前后插入逻辑，以组合的方式实现功能增强。

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    if (UrlUtils.isRegistry(invoker.getUrl())) {
        return protocol.export(invoker);
    }
    return protocol.export(buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER));
}
```

# 配置规范
规范 | 描述
---|---
SPI 配置文件路径 | META-INF/services/、META-INF/dubbo/、META-INF/dubbo/internal/
SPI 配置文件名称 | 接口的全限定名
SPI 配置文件内容 | key=value 形式，多个实现类使用换行符分隔

# 特性
Dubbo 的扩展类一共包含四种特性：自动包装、自动加载、自适应和自动激活。

## 自动包装
在使用 ExtensionLoader 加载扩展类时，如果发现这个扩展类**包含其他扩展点作为构造函数的参数**，则这个扩展类会被认为是一个 Wrapper 类。典型的比如 ProtocolFilterWrapper 扩展类。

```java
public class ProtocolFilterWrapper implements Protocol {

    private final Protocol protocol;

    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
}
```

ProtocolFilterWrapper 虽然实现了 Protocol 接口，但是其构造函数又注入了一个 Protocol 类型的参数，因此它会被认为是一个 Wrapper 类，类似于装饰器模式，把通用的抽象逻辑进行封装或对子类进行增强，让子类可以更加专注于具体的实现。

## 自动加载
除了在构造函数中传入其他扩展实例，我们还经常使用 setter 方法来设置属性值。如果某个扩展类是另外一个扩展点类的成员属性，并且该属性还拥有 setter 方法，那么 Dubbo 会自动注入对应的扩展点实例。ExtensionLoader 在执行扩展点初始化的时候，会自动通过 setter 方法注入对应的实例，这里有一个问题，如果扩展类属性是一个接口，他有多种实现，那么应该注入哪个呢？这就涉及到了 Dubbo SPI 扩展类的第三个特性——自适应。

## 自适应
使用 `@Adaptive` 注解，可以动态地通过 URL 中的参数来确定要使用哪个具体的实现类，从而解决自动加载中的实例注入问题。比如在 Transporter 接口中：

```java
@SPI("netty")
public interface Transporter {
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException;

    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
}
```

这里的 `@Adaptive` 注解中传入了两个参数，分别为 server 和 transporter，在外部调用 Transporter#bind 方法时，会动态地从传入的参数 URL 中提取 key 为 server 的 value 值，如果无法匹配则会继续提取 key 为 transporter 的 value 值。只有在都没有匹配时，才会使用 `@SPI` 注解中的默认值去匹配，此时如果无法匹配就会抛出异常。

## 自动激活
通过自适应的方式来寻找实现类会比较灵活，但是只能激活一个具体的实现类，如果需要激活多个实现类，或者需要根据不同的条件同时激活多个实现类，这就需要使用自动激活特性。使用 `@Activate` 注解，可以标记对应的扩展点默认被激活启用，该注解可以通过传入不同的参数，设置扩展点在不同的条件下被自动激活。

# 扩展点注解
目前能够使用的扩展点注解有三种：`@SPI`、`@Adaptive` 和 `@Activate`。

## `@SPI`
`@SPI` 注解一般用在接口上，它的主要作用就是标记这个接口是一个 Dubbo SPI 接口，即是一个扩展点，可以有多个不同的内置或者用户自定义的实现。该注解可以传入一个 String 类型的参数，表示该接口的默认实现类，比如 Transporter 接口的传入的参数为 netty，对应的配置为：`netty=org.apache.dubbo.remoting.transport.netty4.NettyTransporter`。

```java
@SPI("netty")
public interface Transporter {
  ...
}
```

## `@Adaptive`
`@Adaptive` 注解可以标记在类、接口、枚举类和方法上，但是整个 Dubbo 框架中，只有很少的几个地方使用在类上，比如 AdaptiveExtensionFactory 和 AdaptiveCompiler，其余都标注在方法上。方法级别的注解在第一次使用 ExtensionLoader#getExtension 方法时，**会自动生成和编译一个动态的 Adaptive 类**，从而达到动态指定实现类的效果。

比如 Transporter 接口在 bind 和 connect 方法上都使用了该注解，在初始化扩展点时，会生成一个 Transporter$Adaptive 类，其中会实现这两个方法，方法中会通过 URL 找到并调用真正的实现类。下面的源代码可以使用阿里开源的 Java 诊断工具 [Arthas](https://github.com/alibaba/arthas) 在 Dubbo 服务运行时通过反编译获得。

```java
public class Transporter$Adaptive implements Transporter {
    @Override
    public Client connect(URL uRL, ChannelHandler channelHandler) throws RemotingException {
        if (uRL == null) {
            throw new IllegalArgumentException("url == null");
        }
        URL uRL2 = uRL;
        String string = uRL2.getParameter("client", uRL2.getParameter("transporter", "netty"));
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.remoting.Transporter) name from url (").append(uRL2.toString()).append(") use keys([client, transporter])").toString());
        }
        Transporter transporter = ExtensionLoader.getExtensionLoader(Transporter.class).getExtension(string);
        return transporter.connect(uRL, channelHandler);
    }

    @Override
    public RemotingServer bind(URL uRL, ChannelHandler channelHandler) throws RemotingException {
        if (uRL == null) {
            throw new IllegalArgumentException("url == null");
        }
        URL uRL2 = uRL;
        String string = uRL2.getParameter("server", uRL2.getParameter("transporter", "netty"));
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.remoting.Transporter) name from url (").append(uRL2.toString()).append(") use keys([server, transporter])").toString());
        }
        Transporter transporter = ExtensionLoader.getExtensionLoader(Transporter.class).getExtension(string);
        return transporter.bind(uRL, channelHandler);
    }
}
```

当 `@Adaptive` 注解放在实现类上，则整个实现类会直接作为默认实现，不会再自动生成 Adaptive 类。在扩展点初始化时，如果发现实现类上有 `@Adaptive` 注解，那么会将该类直接赋值给 `cachedAdaptiveClass`，后续实例化的时候，就不会再动态生成代码，而是直接实例化并缓存到 `cachedAdaptiveInstance` 中。在扩展点接口的多个实现类中，只能有一个实现类上可以添加 `@Adaptive` 注解。

另外，如果包装类没有使用 `@Adaptive` 注解指定 key 值，也没有填写 `@SPI` 注解的默认值，那么 Dubbo 会自动把接口名称根据驼峰分开，并用 `.` 符号连接，以此来作为默认实现类的名称，比如 `org.apache.dubbo.xxx.YyyInvokerWrapper` 中的 YyyInvokerWrapper 会被转换成 `yyy.invoker.wrapper`。

## `@Activate`
`@Activate` 注解可以标记在类、接口、枚举类和方法上，主要用在有多个扩展点实现、需要根据不同条件激活多个实现的场景，比如 Filter。`@Activate` 注解可以传入的参数有很多。

参数 | 描述
---|---
String[] group() | URL 中的分组，如果匹配则激活，可以设置多个
String[] value() | URL 中含有该 key 则激活
String[] before() | 扩展点列表，表示哪些扩展点要在本扩展点之前
String[] after() | 扩展点列表，表示哪些扩展点要在本扩展点之后
int order() | 排序

