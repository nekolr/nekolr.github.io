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

# ExtensionLoader
ExtensionLoader 是整个扩展机制的主要逻辑类，通过它可以实现配置（META-INF 目录下的配置文件）的加载、扩展类和实例的缓存、自适应对象的生成等。

ExtensionLoader 的逻辑入口可以分为三个：getExtension、getAdaptiveExtension 和 getActivateExtension，分别用来获取普通的扩展类实例、自适应扩展类实例和自动激活的扩展类实例列表。

## getExtension
大概的流程就是先从 cachedInstances 缓存中获取类实例，如果缓存中没有就先通过配置文件加载扩展类，然后实例化对应的扩展类并放入缓存，然后再处理 setter 依赖注入和 Wrapper 的构造器注入，最后根据是否实现了 LifeCycle 接口决定是否调用扩展类的 initialize 方法。

```java
public T getExtension(String name) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    // 如果扩展名称为 true，则使用 SPI 注解上默认的扩展
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    // 先从普通扩展类实例缓存 cachedInstances 中获取
    final Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    // 缓存中没有则需要创建扩展类实例
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建扩展类实例
                instance = createExtension(name);
                // 设置缓存
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

```java
private T createExtension(String name) {
    // 先从普通扩展类缓存 cachedClasses 中获取
    // 这里的 getExtensionClasses 方法会在初次调用时通过配置文件加载扩展类
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        // 从缓存中获取类实例
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        // 缓存为空则直接实例化后放入缓存
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 依赖注入（setter 注入）
        injectExtension(instance);
        // 通过构造器注入，并实例化 Wrapper 类
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        // 初始化扩展类实例，如果扩展类实现了 org.apache.dubbo.common.context.LifeCycle 接口，
        // 则调用 initialize 初始化方法 
        initExtension(instance);
        // 返回实例
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

## getAdaptiveExtension
大概的流程就是先从 cachedAdaptiveInstance 缓存中获取实例，如果缓存中没有就先通过配置文件加载扩展类，如果扩展类上使用了 `@Adaptive` 注解则该扩展类为默认实现，否则就使用代码生成器生成类似 Transporter$Adaptive 这种的自适应扩展类，拿到 Class 之后就进行实例化，然后进行依赖注入，最终放入缓存并返回。

```java
public T getAdaptiveExtension() {
    // 先从缓存中获取
    Object instance = cachedAdaptiveInstance.get();
    // 缓存中没有则创建自适应扩展类实例
    if (instance == null) {
        if (createAdaptiveInstanceError != null) {
            throw new IllegalStateException("Failed to create adaptive instance: " +
                    createAdaptiveInstanceError.toString(),
                    createAdaptiveInstanceError);
        }

        synchronized (cachedAdaptiveInstance) {
            instance = cachedAdaptiveInstance.get();
            if (instance == null) {
                try {
                    // 创建自适应扩展类实例
                    instance = createAdaptiveExtension();
                    // 放入缓存
                    cachedAdaptiveInstance.set(instance);
                } catch (Throwable t) {
                    createAdaptiveInstanceError = t;
                    throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                }
            }
        }
    }
    return (T) instance;
}
```

```java
private T createAdaptiveExtension() {
    try {
        // 1 getAdaptiveExtensionClass 用来获取或者生成自适应扩展类
        // 2 实例化
        // 3 进行依赖注入
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```

```java
private Class<?> getAdaptiveExtensionClass() {
    // 在初次调用时通过配置文件加载扩展类
    getExtensionClasses();
    // 如果在加载扩展类的时候，扩展类上使用了 @Adaptive 注解，
    // 则该扩展类为默认实现类，会缓存到 cachedAdaptiveClass 中
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    // 创建自适应扩展类
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

```java
private Class<?> createAdaptiveExtensionClass() {
    // 使用代码生成器生成 type$Adaptive 类的代码
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
    // 获取 AdaptiveCompiler 扩展类
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    // 编译代码，默认使用 JavassistCompiler 编译
    return compiler.compile(code, classLoader);
}
```

## getActivateExtension
该方法有多个重载方法，我们关注参数最多的那个即可，大体上就是将缓存过的 `@Activate` 集合根据传入的条件进行筛选，最终使用获取普通扩展类实例的 getExtension 方法获取符合条件的扩展类实例即可。

```java
public List<T> getActivateExtension(URL url, String[] values, String group) {
    List<T> activateExtensions = new ArrayList<>();
    List<String> names = values == null ? new ArrayList<>(0) : asList(values);
    if (!names.contains(REMOVE_VALUE_PREFIX + DEFAULT_KEY)) {
        getExtensionClasses();
        for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
            String name = entry.getKey();
            Object activate = entry.getValue();

            String[] activateGroup, activateValue;
            // 获取注解的参数
            if (activate instanceof Activate) {
                activateGroup = ((Activate) activate).group();
                activateValue = ((Activate) activate).value();
            } else if (activate instanceof com.alibaba.dubbo.common.extension.Activate) {
                activateGroup = ((com.alibaba.dubbo.common.extension.Activate) activate).group();
                activateValue = ((com.alibaba.dubbo.common.extension.Activate) activate).value();
            } else {
                continue;
            }
            // 筛选
            if (isMatchGroup(group, activateGroup)
                    && !names.contains(name)
                    && !names.contains(REMOVE_VALUE_PREFIX + name)
                    && isActive(activateValue, url)) {
                // getExtension 方法用来获取普通的扩展类实例
                activateExtensions.add(getExtension(name));
            }
        }
        activateExtensions.sort(ActivateComparator.COMPARATOR);
    }
    // 以下省略部分代码
    ...
    return activateExtensions;
}
```

## ExtensionFactory
如果我们单独使用 Dubbo 的 SPI 机制，可以先定义接口，然后添加配置文件（META-INF 目录下的配置文件），最后通过 ExtensionLoader 加载。

```java
public class Test {
    public static void main(String[] args) {
        UserService userService = ExtensionLoader.getExtensionLoader(UserService.class)
                .getDefaultExtension();
        userService.getAll();
    }
}
```

可以看到，在使用 ExtensionLoader 的时候，需要先通过 getExtensionLoader 方法获取 ExtensionLoader，在这个方法中会对 ExtensionLoader 进行初始化。

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    // 判断 type 是否是接口
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    // 判断是否有 @SPI 注解
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }
    // 缓存中获取
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        // 缓存中没有就 new 一个
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}

// 私有化的构造函数
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null :
            ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

可以看到 ExtensionLoader 中有一个比较关键的属性：objectFactory，它在 ExtensionLoader 实例化的时候会通过 getAdaptiveExtension 方法获取 ExtensionFactory 接口的实现类。ExtensionFactory 有三个实现类：SpiExtensionFactory、SpringExtensionFactory 和 AdaptiveExtensionFactory，其中 AdaptiveExtensionFactory 类上使用了 `@Adaptive` 注解，因此它是默认的实现类。那么 ExtensionFactory 到底是干什么用的呢？我们可以全局搜索一下，发现它在 injectExtension 方法中被使用。

```java
private T injectExtension(T instance) {
    if (objectFactory == null) {
        return instance;
    }
    try {
        // 遍历所有的方法
        for (Method method : instance.getClass().getMethods()) {
            // 不是 setter 方法就跳过
            if (!isSetter(method)) {
                continue;
            }
            if (method.getAnnotation(DisableInject.class) != null) {
                continue;
            }
            // 获取 setter 方法第一个参数的类型
            Class<?> pt = method.getParameterTypes()[0];
            if (ReflectUtils.isPrimitives(pt)) {
                continue;
            }
            try {
                // 获取 setter 方法对应的属性名称，比如 setVersion，那么属性就是 version
                String property = getSetterProperty(method);
                // 通过 ExtensionFactory 获取对应的实例
                Object object = objectFactory.getExtension(pt, property);
                if (object != null) {
                    // 调用 setter 方法注入属性
                    method.invoke(instance, object);
                }
            } catch (Exception e) {
                logger.error("Failed to inject via method " + method.getName()
                        + " of interface " + type.getName() + ": " + e.getMessage(), e);
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

在获取扩展类实例时，如果扩展类实例的属性包含其他扩展类实例，那么就会通过 ExtensionFactory#getExtension 方法加载，加载的范围包括 **Dubbo 自身缓存的扩展类实例以及 Spring 容器实例**，这也就意味着，我们可以在 Dubbo 的扩展类实例中使用其他扩展类实例和 Spring 容器中的实例。