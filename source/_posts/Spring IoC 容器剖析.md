---
title: Spring IoC 容器剖析
date: 2018/5/27 19:08:0
tags: [Spring]
categories: [Spring]
---

> 使用 spring 5.1.19.RELEASE 分析

整个 IoC 容器都是围绕着 `BeanFactory` 和 `ApplicationContext` 来设计的。`BeanFactory` 提供了容器的基本功能。`ApplicationContext` 继承自 `BeanFactory`，不光实现了容器的基本功能，还实现了一些更高级的容器特性。  

<!--more-->		

# BeanFactory 容器设计

![ConfigurableBeanFactory](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/27/3q7.png)

这是一条主要的 `BeanFactory` 设计路线。

- `BeanFactory` 提供了如 `getBean` 这样的基本方法用来获取容器中的 bean。
- `HierarchicalBeanFactory` 提供了 `getParentBeanFactory` 方法，使得容器获得了父子容器管理的功能。
- `SingletonBeanRegistry` 提供了管理单例 bean 的功能。
- `ConfigurableBeanFactory` 提供了一些容器的配置功能，比如设置容器的父容器的 `setParentBeanFactory` 方法。

在这条路线上，有一个实现了基本容器功能的类 `DefaultListableBeanFactory`。

![DefaultListableBeanFactory](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/27/o6x.png)

`DefaultListableBeanFactory` 已经实现了容器的基本功能，也就是说我们可以直接使用它了。该类还有一个子类 `XmlBeanFactory`，不过已经不推荐使用了。  

```java
@Deprecated
@SuppressWarnings({"serial", "all"})
public class XmlBeanFactory extends DefaultListableBeanFactory {
    private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

    public XmlBeanFactory(Resource resource) throws BeansException {
      this(resource, null);
    }
    public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
      super(parentBeanFactory);
      this.reader.loadBeanDefinitions(resource);
    }
}
```

通过查看 `XmlBeanFactory` 的代码，我们可以模仿它来使用 `DefaultListableBeanFactory`。

```java
public static void main(String[] args) {
    // 定义资源文件
    ClassPathResource classPathResource = new ClassPathResource("applicationContext.xml");
    // 创建 DefaultListableBeanFactory 实例
    DefaultListableBeanFactory defaultListableBeanFactory = new DefaultListableBeanFactory();
    // 创建 XmlBeanDefinitionReader
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(defaultListableBeanFactory);
    // 使用 XmlBeanDefinitionReader 来解析资源文件中的 bean 定义
    reader.loadBeanDefinitions(classPathResource);
    // 获取容器中的 bean
    ElectricCar electricCar = (ElectricCar) defaultListableBeanFactory.getBean("electricCar");
}
```

# ApplicationContext 容器设计
IoC 容器的第二条设计路线以 `ApplicationContext` 接口为主，在继承了 `BeanFactory` 的同时，还继承了 `MessageSource`、`ApplicationEventPublisher`、`ResourceLoader` 等接口。

![ApplicationContext](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/27/WnD.png)

 `ApplicationContext` 作为 `BeanFactory` 的实现，和 `XmlBeanFactory` 一样，也是在 `DefaultListableBeanFactory` 这个基本的容器实现上做扩展（一般是通过持有一个 `DefaultListableBeanFactory` 对象来实现）。我们常用的应用上下文基本上都是 `ConfigurableApplicationContext` 或者 `WebApplicationContext` 的实现。

 ![ConfigurableWebApplicationContext](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/27/YAN.png)

实现了 ConfigurableApplicationContext 接口的类 | 描述
---|---
FileSystemXmlApplicationContext | 通过文件系统加载 xml 配置文件来初始化容器
ClassPathXmlApplicationContext | 通过 classpath 加载 xml 配置文件来初始化容器
AnnotationConfigApplicationContext | 通过加载纯注解的配置类来初始化容器

实现了 WebApplicationContext 接口的类 | 描述
---|---
AnnotationConfigWebApplicationContext | 通过加载纯注解的配置类来初始化具有 Web 功能的容器
XmlWebApplicationContext | 在 Web 应用中，我们一般会在 web.xml 中设置一个监听器：ContextLoaderListener，默认情况下，它会在 Servlet 容器启动时创建并初始化一个 XmlWebApplicationContext

# 容器初始化过程
IoC 容器的初始化是通过 `AbstractApplicationContext` 的 `refresh` 方法来完成的，这个过程主要包括 BeanDefinition 的 Resource 定位、载入和注册三个基本操作，Spring 将这三个过程分开，使用了不同的模块来完成（比如使用 ResourceLoader、BeanDefinitionReader 等）。通过这种方式，可以让用户根据需要更加灵活地对这三个过程进行修改或者扩展。

![AbstractApplicationContext](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/28/LXL.png)

## 包含 BeanDefinition 的资源定位
我们知道，BeanDefinition 接口是 Bean 定义的抽象，用来描述 Bean 的一些信息，比如是单例 Bean 还是原型 Bean，对应的类的全限定名称，是否需要延迟初始化，依赖关系等等。Bean 定义的存在形式有很多种，常见的可能会在文件系统中或者在类路径中，因此，Spring 抽象出了 Resource 接口，在文件系统中的 Bean 定义资源就可以通过 FileSystemResource 来抽象，而在类路径的 Bean 定义资源则可以通过 ClassPathResource 来抽象，然后它们都可以通过对应的 ResourceLoader 来获取。

在以编程的方式使用 DefaultListableBeanFactory 时，我们需要定义一个 Resource 来定位容器所使用的 Bean 定义资源，这里的 Resource 并不能直接交给 DefaultListableBeanFactory 使用，因为 DefaultListableBeanFactory 只是一个单纯的 IoC 容器，我们需要为它配置一个特定的 BeanDefinitionReader，用来读取 Bean 定义资源，并将 Bean 定义转换成容器能够处理的形式。而我们使用的很多 ApplicationContext 实现中已经提供了一系列能够加载不同 Resource 的读取器实现，我们以 FileSystemXmlApplicationContext 为例，从头到尾梳理一下容器的初始化过程。

![FileSystemXmlApplicationContext](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202104211444/2021/04/12/5NM.png)

通过上图可以看出，FileSystemXmlApplicationContext 通过继承 AbstractApplicationContext，具备了 ResourceLoader 读取 Resource 的能力。

```java
public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {
    public FileSystemXmlApplicationContext() {
    }
    // 这个构造函数可以传入父容器
    public FileSystemXmlApplicationContext(ApplicationContext parent) {
      super(parent);
    }
    // 这个构造函数可以传入 BeanDefinition 所在的文件路径
    public FileSystemXmlApplicationContext(String configLocation) throws BeansException {
      this(new String[] {configLocation}, true, null);
    }

    // 省略部分代码

    public FileSystemXmlApplicationContext(
        String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
        throws BeansException {
      super(parent);
      // 设置 BeanDefinition 所在文件路径
      setConfigLocations(configLocations);
      if (refresh) {
        // 这个 refresh 启动了容器的初始化过程，当然也包括 BeanDefinition 的载入过程
        refresh();
      }
    }
    @Override
    protected Resource getResourceByPath(String path) {
      if (path.startsWith("/")) {
        path = path.substring(1);
      }
      return new FileSystemResource(path);
    }
}
```

在最后这个构造器中调用了父类的构造器，我们沿着类图一直向上，能够追溯到 AbstractApplicationContext 类：

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    public AbstractApplicationContext() {
      this.resourcePatternResolver = getResourcePatternResolver();
    }
    public AbstractApplicationContext(@Nullable ApplicationContext parent) {
      this();
      setParent(parent);
    }
    protected ResourcePatternResolver getResourcePatternResolver() {
      return new PathMatchingResourcePatternResolver(this);
    }
}
```

可以看到，在 AbstractApplicationContext 中初始化了一个 PathMatchingResourcePatternResolver 类，它实现了 ResourcePatternResolver 接口。我们知道，Resource 是资源的抽象，通过它可以访问各种包含 Bean 定义的资源，但是该接口有一个问题：它不支持以通配符的方式读取资源。如果我们要访问同一个路径下所有符合条件的资源，只能将读取的资源路径全部写出来才可以，ResourcePatternResolver 的出现就是为了解决这个问题的，它能够按照相应的模式匹配策略将资源路径转换成对应的资源，默认情况下使用的 PathMatchingResourcePatternResolver 是按照 Ant 风格的匹配策略来处理资源路径的。

```java
public void setConfigLocations(@Nullable String... locations) {
  if (locations != null) {
    Assert.noNullElements(locations, "Config locations must not be null");
    this.configLocations = new String[locations.length];
    for (int i = 0; i < locations.length; i++) {
      // resolvePath 方法主要用来处理资源路径中的占位符
      this.configLocations[i] = resolvePath(locations[i]).trim();
    }
  }
  else {
    this.configLocations = null;
  }
}

protected String resolvePath(String path) {
    // 处理路径中的占位符
    return getEnvironment().resolveRequiredPlaceholders(path);
}
```

我们接着回到 setConfigLocations 方法。在该方法中，resolvePath 方法能够将资源路径字符串中的占位符转换成对应的值。比如资源路径为：`classpath:applicationContext-${profile}.xml`，那么该方法会从环境变量（包括系统变量、用户自定义的变量等）中寻找对应的值进行替换。

对于容器的启动来说，refresh 方法是一个很重要的方法，它详细地描述了整个 ApplicationContext 的初始化过程，比如 BeanFactory 的刷新，MessageSource 和 PostProcessor 的注册等等，这个执行过程为 Bean 的生命周期管理创造了条件。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        /**
        *   Prepare this context for refreshing.
        *   准备要刷新的上下文。
        */
        prepareRefresh();
        /**
        *   Tell the subclass to refresh the internal bean factory.
        *   告诉子类刷新内部的 BeanFactory。
        */
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        /**
        *   Prepare the bean factory for use in this context.
        *   对 BeanFactory 做相关配置，如设置 BeanPostProcessor、BeanClassLoader 等。
        */
        prepareBeanFactory(beanFactory);
        try {
            /**
            *   Allows post-processing of the bean factory in context subclasses.
            *   允许在上下文子类中对 BeanFactory 进行后处理。
            *   
            *   protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {}
            *   该方法是一个空方法，由 AbstractApplicationContext 的子类去实现。
            */
            postProcessBeanFactory(beanFactory);
            /**
            *   Invoke factory processors registered as beans in the context.
            *   执行在上下文注册为 bean 的 BeanFactoryPostProcessor。
            *   
            *   Spring 提供的扩展点，我们可以实现 BeanFactoryPostProcessor 接口，并注册为 Spring 的 bean。
            *   在本方法中表现为通过将读取到的 beanDefinitionNames 逐个解析，判断是否为 BeanFactoryPostProcessor 
            *   的实现类来确定执行哪些 BeanFactoryPostProcessor。
            *   
            *   这样我们就可以在 bean 实例化之前，读取 BeanFactory 中的 bean 配置，并根据需要进行修改，
            *   比如修改 bean 的 scope 等等，甚至可以预先将 bean 初始化（不推荐）。
            */  
            invokeBeanFactoryPostProcessors(beanFactory);
            /** 
            *   Register bean processors that intercept bean creation.
            *   注册拦截 bean 创建的处理器
            *   
            *   Spring 提供的扩展点，我们可以实现 BeanPostProcessor 接口，并注册为 Spring 的 bean。
            *   Spring 发现这些实现类的方式和发现 BeanFactoryPostProcessor 的方式一致。
            *
            *   实现 BeanPostProcessor 接口需要重写 postProcessBeforeInitialization() 和 
            *   postProcessAfterInitialization() 方法，这两个方法会在每个非手工注册的 bean 初始化之前和初始化之后执行。
            */ 
            registerBeanPostProcessors(beanFactory);
            /**
            *   Initialize message source for this context.
            *   初始化消息国际化。
            */
            initMessageSource();
            /**
            *   Initialize event multicaster for this context.
            *   初始化事件广播。
            */
            initApplicationEventMulticaster();
            /**
            *   Initialize other special beans in specific context subclasses.
            *   这是一个模版方法，允许子类在进行 bean 初始化之前进行一些定制操作。默认空实现。
            */
            onRefresh();
            /**
            *   Check for listener beans and register them.
            *   注册事件监听者（实现了 ApplicationListener 接口）。
            */
            registerListeners();
            /**
            *   Instantiate all remaining (non-lazy-init) singletons.
            *   完成 BeanFactory 的最终配置，并实例化所有剩下的非懒加载的单例 bean。
            */
            finishBeanFactoryInitialization(beanFactory);
            /**
            *   Last step: publish corresponding event.
            *   完成刷新，调用 LifecycleProcessor 的 onRefresh() 方法，发布 ContextRefreshedEvent 事件
            */
            finishRefresh();
        }
        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }
            /**
            *   Destroy already created singletons to avoid dangling resources.
            *   销毁已经创建的单例 bean
            */
            destroyBeans();
            // Reset 'active' flag.
            cancelRefresh(ex);
            // Propagate exception to caller.
            throw ex;
        }
        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

在这里，我们需要重点关注 obtainFreshBeanFactory 方法。

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 刷新 BeanFactory
    refreshBeanFactory();
    // 获取 BeanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

```java
/**
*   org.springframework.context.support.AbstractRefreshableApplicationContext
*/
@Override
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        // 销毁已经实例化的 Bean
        destroyBeans();
        // 关闭 BeanFactory
        closeBeanFactory();
    }
    try {
        // 默认创建一个 DefaultListableBeanFactory，如果有父容器会作为参数传入
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        // 此处加载 bean 定义信息
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

Spring 在刷新 BeanFactory 时，会检测 BeanFactory 是否已经创建，已经创建会执行销毁方法，然后重新创建。在 BeanFactory 创建完成后，会开始加载并解析 bean 的配置。

```java
/**
*  org.springframework.context.support.AbstractXmlApplicationContext
*/
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 创建 XmlBeanDefinitionReader 来读取和解析 BeanDefinition
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    // 配置读取器的上下文环境
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    // 允许子类自定义读取器的初始化方法
    initBeanDefinitionReader(beanDefinitionReader);
    // 真正的加载 BeanDefinition
    loadBeanDefinitions(beanDefinitionReader);
}

protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    // 使用 ClassPathXmlApplicationContext 容器时会调用此处
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
      reader.loadBeanDefinitions(configResources);
    }
    // 使用 FileSystemXmlApplicationContext 容器时调用此处
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
      reader.loadBeanDefinitions(configLocations);
    }
}
```

我们接着追溯 loadBeanDefinitions 方法，会发现由 AbstractBeanDefinitionReader 调用了该方法：

```java
/**
* org.springframework.beans.factory.support.AbstractBeanDefinitionReader
*/
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    // 获取 ResourceLoader
    ResourceLoader resourceLoader = getResourceLoader();
    if (resourceLoader == null) {
      throw new BeanDefinitionStoreException(
          "Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
    }
    // 如果这个 ResourceLoader 同样实现了 ResourcePatternResolver
    // 很多容器都实现了该接口，方便通过模式匹配加载多个资源
    if (resourceLoader instanceof ResourcePatternResolver) {
      try {
        // 获取匹配到的资源
        Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
        int count = loadBeanDefinitions(resources);
        if (actualResources != null) {
          Collections.addAll(actualResources, resources);
        }
        if (logger.isTraceEnabled()) {
          logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
        }
        return count;
      }
      catch (IOException ex) {
        throw new BeanDefinitionStoreException(
            "Could not resolve bean definition resource pattern [" + location + "]", ex);
      }
    }
    else {
      // 只能通过访问绝对路径的方式加载单个资源
      // Can only load single resources by absolute URL.
      Resource resource = resourceLoader.getResource(location);
      int count = loadBeanDefinitions(resource);
      if (actualResources != null) {
        actualResources.add(resource);
      }
      if (logger.isTraceEnabled()) {
        logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
      }
      return count;
    }
}

public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int count = 0;
    for (Resource resource : resources) {
      count += loadBeanDefinitions(resource);
    }
    return count;
}
```

在该方法中，如果 ResourceLoader 同样实现了 ResourcePatternResolver 接口，那么就通过模式匹配的方式加载资源；否则就单纯的使用绝对路径来加载单个资源。至此，所有的资源都已经完成了定位，接下来开始调用 loadBeanDefinitions 方法进行 BeanDefinition 的载入和解析。

## BeanDefinition 的载入和解析
在这一过程中，首先需要读取所有定位好的资源，接着要按照一定的规则将读取到的内容转换成 IoC 容器内部的数据结构，这个数据结构对应的就是 BeanDefinition 接口。

我们接着分析上面的代码，在 loadBeanDefinitions 方法中，循环调用了一个同名的 loadBeanDefinitions 方法，该方法只有一个 Resource 参数，由 AbstractBeanDefinitionReader 的子类来实现，在我们的这个例子当中，显然这里调用的是 XmlBeanDefinitionReader 的实现。

```java
/**
* org.springframework.beans.factory.xml.XmlBeanDefinitionReader
*/
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isTraceEnabled()) {
      logger.trace("Loading XML bean definitions from " + encodedResource);
    }

    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
      currentResources = new HashSet<>(4);
      this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
      throw new BeanDefinitionStoreException(
          "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
      // 这里拿到资源对应的流，封装成 InputSource
      InputStream inputStream = encodedResource.getResource().getInputStream();
      try {
        InputSource inputSource = new InputSource(inputStream);
        if (encodedResource.getEncoding() != null) {
          inputSource.setEncoding(encodedResource.getEncoding());
        }
        // 真正的加载流程
        return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
      }
      finally {
        inputStream.close();
      }
    }
    catch (IOException ex) {
      throw new BeanDefinitionStoreException(
          "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
      currentResources.remove(encodedResource);
      if (currentResources.isEmpty()) {
        this.resourcesCurrentlyBeingLoaded.remove();
      }
    }
}

protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    throws BeanDefinitionStoreException {
    try {
      // 读取文档的内容
      Document doc = doLoadDocument(inputSource, resource);
      // 注册 Bean 定义
      int count = registerBeanDefinitions(doc, resource);
      if (logger.isDebugEnabled()) {
        logger.debug("Loaded " + count + " bean definitions from " + resource);
      }
      return count;
    }
    catch (BeanDefinitionStoreException ex) {
      throw ex;
    }
    // 省略部分代码
}
```

真正执行载入 Bean 定义的是 doLoadBeanDefinitions 方法。在该方法中，doLoadDocument 方法只负责读取 xml 文档内容，生成一个 Document 对象。

```java
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
    return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
        getValidationModeForResource(resource), isNamespaceAware());
}

protected int getValidationModeForResource(Resource resource) {
    // 获取 XML 文档的校验方式
    int validationModeToUse = getValidationMode();
    if (validationModeToUse != VALIDATION_AUTO) {
      return validationModeToUse;
    }
    // 试图查找 XML 文档的校验方式
    int detectedMode = detectValidationMode(resource);
    if (detectedMode != VALIDATION_AUTO) {
      return detectedMode;
    }
    // Hmm, we didn't get a clear indication... Let's assume XSD,
    // since apparently no DTD declaration has been found up until
    // detection stopped (before finding the document's root tag).
    return VALIDATION_XSD;
}

protected int detectValidationMode(Resource resource) {
    if (resource.isOpen()) {
      // 省略部分代码
    }

    InputStream inputStream;
    try {
      inputStream = resource.getInputStream();
    }
    catch (IOException ex) {
      // 省略部分代码
    }

    try {
      // 根据文件头部是否包含 DOCTYPE 等关键字来判断采用哪种校验方式
      return this.validationModeDetector.detectValidationMode(inputStream);
    }
    catch (IOException ex) {
      throw new BeanDefinitionStoreException("Unable to determine validation mode for [" +
          resource + "]: an error occurred whilst reading from the InputStream.", ex);
    }
}
```

在 doLoadDocument 方法中，我们可以关注一下 getValidationModeForResource 这个方法，该方法主要用来获取应该采用何种方式校验 XML 文件。如果 XML 文件头部包含 `DOCTYPE` 等关键字，那么会通过 DTD 约束来校验文件；否则会通过 XSD 约束来校验文件。使用了 DTD 约束的 XML 配置文件类似下面这样：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN" "http://www.springframework.org/dtd/spring-beans-2.0.dtd">
<beans></beans>
```

与此同时，还有一个方法值得关注，那就是 getEntityResolver，该方法用来获取 EntityResolver。我们知道，配置文件在头部会定义约束文件，而约束文件一般都带有 URL 地址，这意味着我们需要网络来获取约束文件，但现实环境往往比较复杂，项目有时会面临离线的状态。为了解决这个问题，约束文件一般会跟随框架一起打包，那么这些约束文件被放在哪里了呢？在 Spring 中，它们的位置被记录在了 `META-INF/spring.schemas` 文件中。比如我们打开 spring-beans.jar 下的该文件：

```
http\://www.springframework.org/schema/beans/spring-beans-2.0.xsd=org/springframework/beans/factory/xml/spring-beans.xsd
http\://www.springframework.org/schema/beans/spring-beans-2.5.xsd=org/springframework/beans/factory/xml/spring-beans.xsd
http\://www.springframework.org/schema/beans/spring-beans-3.0.xsd=org/springframework/beans/factory/xml/spring-beans.xsd
# 以下省略
```

可以看到，各个版本以及没有版本号的约束文件，都指向了同一路径下的一个约束文件。在 EntityResolver 接口中，只有一个 resolveEntity 方法，这个方法的作用就是根据提供的约束文件 URL，寻找对应的离线约束文件。

我们接着回到 doLoadBeanDefinitions 方法。此时我们已经将所有的配置转换成了一个 `org.w3c.dom.Document` 结构，接下来只需要将它转换成对应的 BeanDefinitions 就可以了。

```java
/**
* org.springframework.beans.factory.xml.XmlBeanDefinitionReader
*/
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 创建 BeanDefinitionDocumentReader
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    // 将解析文档的工作委派给它
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}

public XmlReaderContext createReaderContext(Resource resource) {
    return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
        this.sourceExtractor, this, getNamespaceHandlerResolver());
}
```

在 registerBeanDefinitions 方法中，又创建了一个 BeanDefinitionDocumentReader，然后将文档解析的工作委派给了它，同时还给它额外传入了一个 ReaderContext 参数，这个参数我们可以理解为一个文档读取的上下文环境。在这个上下文环境当中，有一个重要的属性：NamespaceHandlerResolver，乍一看这个名字起的有点奇怪，Handler 后面为什么还加了个 Resolver？我们带着这个问题，查看 getNamespaceHandlerResolver 方法，发现在这里最终是创建了一个默认的实现：DefaultNamespaceHandlerResolver，它的作用是从 `META-INF/spring.handlers` 文件中加载不同命名空间所对应的 NamespaceHandler。比如 spring-context.jar 下该文件的内容是这样的：

```
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler
```

在这里我们随便打开一个 NamespaceHandler，比如 ContextNamespaceHandler：

```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {
  @Override
  public void init() {
    registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
    registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
    registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
    registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
    registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
    registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
    registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
    registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
  }
}
```

可以看到，这里初始化的是 context 命名空间中所有可用的元素（标签）和它们对应的解析器，这个解析器被 Spring 抽象为 BeanDefinitionParser 接口。我们常见的一些命名空间配置，比如 `<context:component-scan base-package="xxx" />` 等，就由这些解析器来处理。

我们回到之前的代码。由于文档解析的工作交给了 BeanDefinitionDocumentReader，同时在这里使用的是它的一个默认实现：DefaultBeanDefinitionDocumentReader，因此我们继续深入到它的方法中。

```java
/**
*  org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader
*/
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    doRegisterBeanDefinitions(doc.getDocumentElement());
}

protected void doRegisterBeanDefinitions(Element root) {
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);
    // 判断是否是默认的命名空间：http://www.springframework.org/schema/beans
    if (this.delegate.isDefaultNamespace(root)) {
      // 获取 beans 的 profile 属性
      String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
      if (StringUtils.hasText(profileSpec)) {
        String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
            profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        // We cannot use Profiles.of(...) since profile expressions are not supported
        // in XML config. See SPR-12458 for details.
        // 从 ReadContext 的环境变量中获取当前被激活的 profile
        // 如果当前正在解析的配置文件不是被激活的 profile，那么直接跳过文档解析的过程
        if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
          if (logger.isDebugEnabled()) {
            logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                "] not matching: " + getReaderContext().getResource());
          }
          return;
        }
      }
    }
    // 前置处理，默认是一个空实现，可以由子类来重写
    preProcessXml(root);
    // 解析 BeanDefinitions
    parseBeanDefinitions(root, this.delegate);
    // 后置处理，默认是一个空实现，可以由子类来重写
    postProcessXml(root);

    this.delegate = parent;
}
```

在该方法中，首先执行了一些前置工作，比如检查默认激活的 profile 与当前解析的文档是否一致，如果不一致则直接跳过解析过程。然后还有两个可以由子类扩展的前置和后置处理的方法，以及真正执行解析工作的 parseBeanDefinitions 方法。

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    // 判断是否是默认的命名空间
    if (delegate.isDefaultNamespace(root)) {
      NodeList nl = root.getChildNodes();
      for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (node instanceof Element) {
          Element ele = (Element) node;
          // 如果是默认的命名空间，则使用默认的元素处理方法
          if (delegate.isDefaultNamespace(ele)) {
            parseDefaultElement(ele, delegate);
          }
          else {
            delegate.parseCustomElement(ele);
          }
        }
      }
    }
    else {
      // 不是默认的命名空间则执行自定义元素处理方法
      delegate.parseCustomElement(root);
    }
}

private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    // 如果是 import 标签，则执行 import 的逻辑
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
      importBeanDefinitionResource(ele);
    }
    // 如果是 alias 标签，则处理 alias 的逻辑
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
      processAliasRegistration(ele);
    }
    // 如果是 bean 标签，则执行解析 BeanDefinition 的逻辑
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
      processBeanDefinition(ele, delegate);
    }
    // 如果是 beans 标签，则通过递归的方式回到上面的方法
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
      // 递归
      doRegisterBeanDefinitions(ele);
    }
}

public BeanDefinition parseCustomElement(Element ele) {
    return parseCustomElement(ele, null);
}

public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
    // 获取命名空间约束文件 URI
    String namespaceUri = getNamespaceURI(ele);
    if (namespaceUri == null) {
      return null;
    }
    // 获取对应的命名空间处理器
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
      error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
      return null;
    }
    // 将解析的工作交给命名空间处理器
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

parseBeanDefinitions 方法的逻辑很清晰：根据不同的命名空间，调用不同的处理逻辑。在默认命名空间中有四类标签：`import`、`alias`、`bean` 和 `beans`，其中 `bean` 标签对应的处理逻辑能够将 bean 定义转换成 BeanDefinition。对于其他命名空间的标签，我们在前面提到过，使用 NamespaceHandlerResolver 可以获取对应的命名空间处理器，然后将解析工作交给对应的 NamespaceHandler 来处理。这里我们只重点关注 `bean` 标签的处理。

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 解析标签，并将转换后的 BeanDefinition 放入 BeanDefinitionHolder 中
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
      bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
      try {
        // 注册 BeanDefinition
        BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
      }
      catch (BeanDefinitionStoreException ex) {
        getReaderContext().error("Failed to register bean definition with name '" +
            bdHolder.getBeanName() + "'", ele, ex);
      }
      // Send registration event.
      getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

在该方法中，parseBeanDefinitionElement 方法将解析 `bean` 标签并生成对应的 BeanDefinition（创建的是一个 GenericBeanDefinition），最后将它放入一个 BeanDefinitionHolder 中。接下来的需要做的就是将 BeanDefinition 注册到容器中。

## BeanDefinition 的注册
向 IoC 容器注册 BeanDefinition 是通过调用 BeanDefinitionRegistry 接口的实现来完成的。简单来说就是在 IoC 容器内部有一个名为 beanDefinitionMap 的 ConcurrentHashMap，注册的过程就是将 BeanDefinition 放入这个 Map 中。

```java
/**
*  org.springframework.beans.factory.support.BeanDefinitionReaderUtils
*/
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {

    // 注册 BeanDefinition
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
    // 注册别名
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
      for (String alias : aliases) {
        registry.registerAlias(beanName, alias);
      }
    }
}
```

# 循环依赖的问题
这部分内容之所以放在容器的依赖注入之前，是因为将这类问题单独拎出来，再结合 Spring 对于循环依赖问题的处理完整分析一遍，对于接下来解析 Spring 依赖注入部分的代码很有帮助。

所谓循环依赖，说白了就是一个或多个对象实例之间存在直接或者间接的依赖关系，这个依赖关系最终形成了一个环形的结构。一般循环依赖可以简化为三种情况，即自己依赖自己，两两之间的直接依赖和多个对象之间的间接依赖。前两种情况的直接循环依赖比较好识别，第三种间接循环依赖的情况有时候因为业务代码调用层级很深，不容易识别出来。

![循环依赖](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202104211444/2021/04/16/kbQ.png)

在 Spring 中，出现循环依赖的场景主要有以下几种：

![Spring 中循环依赖的主要场景](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202104211444/2021/04/16/Evd.png)

在这些可能出现循环依赖的场景中，有些 Spring 是能够处理的，有些则不能。Spring 跟我们一样，我们在平常编码时无法解决的循环依赖问题，它同样也无法解决，比如多例的 setter 注入和构造器注入。我们举个例子，用大白话来分析一下。假设有两个类 A 和 B，它们之间互相依赖：

```java
public class A {
  private B b;
  // 省略构造器和 setter 方法
}

public class B {
  private A a;
  // 省略构造器和 setter 方法
}
```

如果我们要实现多例，也就是每次都要创建一个新的对象。在使用 setter 方法注入时，我们首先需要实例化 A，由于 A 依赖于 B，所以我们又实例化了一个 B，然后使用 setter 方法将 B 的实例注入到 A 的实例中。接着我们又发现 B 依赖于 A，同时由于我们要实现多例，那么此时还需要再重新实例化一个 A，然后将这个 A 的新实例通过 setter 注入给 B 的实例。这还没完，A 的新实例也需要注入呀，因此我们又实例化了一个 B 的新实例，如此往复。这显然是没有尽头的，也是无法处理的。

而在构造器注入的场景下，无法解决循环依赖的问题是因为创建对象需要调用构造器，而构造器需要传入依赖的对象，由于此时被依赖的对象还没有创建，因此同样使用构造器创建，不过该构造器也需要传入依赖的对象，这就陷入了死胡同。

DependsOn 循环依赖则比较特殊，它是 Spring 框架特有的。对于一个 Bean，Spring 大体上向用户提供了两种方式来配置它的依赖。一种是使用 `depends-on` 属性或者 `@DependsOn` 注解，另一种是使用 `ref` 属性等类似的方式。其中，通过 `ref` 属性来配置依赖的对象中需要持有被依赖的对象，而 DependsOn 则不需要。所以我们可以理解为这种依赖只是一种初始化顺序上的依赖，那么因此我们就很容易理解：具有初始化顺序依赖的两个对象之间不应该存在逻辑上（初始化上）的依赖。

到了单例环境中，循环依赖的问题其实是很容易处理的，利用我们经常使用的 setter 方法执行注入即可。

```java
A a = new A();
B b = new B();
a.setB(b);
b.setA(a);
```

如果让我们按照这个思路，实现一个简易的，能够处理循环依赖的 IoC 容器，我们应该怎么做呢？回看上面的代码，你会发现其实这是一种延迟初始化的思想。也就说，我们在初始化 Bean 的时候，首先通过无参构造创建了一个“空”对象，接着我们要将这个对象的引用存储起来，方便我们后续还能找到这个未初始化完成的对象，执行 setter 注入。这里有一个很重要的信息，那就是存储对象的引用，因此我们需要一个缓存容器。到了这里，我们离 Spring 的实现又近了一步。没错，Spring 的设计思路跟我们一样，不过不同的是，它一次性使用了三个这样的缓存容器，也就是我们俗称的三级缓存。

```java
/**
*   单例对象的缓存容器，对应的结构是：bean 名称 -> bean 实例，这个 bean 是完全初始化的
*/
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
/**
*   早期暴露出来的单例对象的缓存容器，对应的结构是：bean 名称 -> bean 实例，这个 bean 是不完全初始化的
*/
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
/**
*   单例工厂的缓存容器，对应的结构是：bean 名称 -> ObjectFactory 实例，它是可以生成 Bean 的工厂
*/
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

一级缓存为 singletonObjects，存储的是已经完全初始化的 bean 实例。二级缓存为 earlySingletonObjects，存储的是已经实例化，但是还没有初始化的 bean 实例。三级缓存为 singletonFactories，存储的是能够生成 bean 实例的工厂对象。Spring 为什么要使用三级缓存来处理循环依赖的问题呢？换句话说，只使用一级缓存，或者只使用二级缓存可以吗？答案是不行。

我们先看只使用一级缓存为什么不行。同样还是拿着 A 和 B 这两个类来举例。用户获取 A 的实例，此时缓存容器中没有，因此 A 要先实例化，实例化之后放入缓存容器中，接着我们需要填充 A 的属性，也就是 B 的实例。此时我们需要先从缓存容器中尝试获取 B 的实例，如果没有我们需要先实例化 B。等等！我们先别急着向下分析，在 A 实例化并放入缓存容器之后，如果此时有个用户尝试获取 A 的实例怎么办？或者尝试获取 B 的实例，同样经过了上述步骤，在填充属性时，从缓存容器中拿到了 A 的实例。在这些场景中，A 的实例都没有完全初始化，但是用户已经获取到并开始准备使用它了。这显然是有问题的，而产生问题的关键是我们无法得知某一时刻缓存容器中的实例是否完全初始化了。比较简单直接的解决方法就是使用两个缓存容器，一个存放实例化但没初始化的 bean，另一个存放实例化并且完全初始化的 bean，各司其职。那为什么我们在一开始又说：在 Spring 中，只使用二级缓存是不行的呢？

我们不要忘了，Spring Framework 作为一个成熟的商业产品，IoC 只是它的一个基本功能。除了 IoC，它还有一个很重要也是很基础的功能：AOP。如果一个 bean 经过了 AOP 的织入（Weaving），那么我们拿到的这个 bean 实际上是一个代理 bean。在我们像上面那样去使用二级缓存时，二级缓存中存放只是原始的 bean 实例，这样其他依赖该 bean 的实例在填充属性时获取到的也只是原始的 bean 实例，并不是我们真正想要获取的代理 bean 实例。那么你可能又会问了，如果我们不像上面那样将实例化后的 bean 对象放入二级缓存，而是将代理对象（代理对象包裹着未初始化的原始实例）放入二级缓存中不就行了吗。理论上来说，这是完全没有问题的，那么 Spring 又为什么非要使用三级缓存呢？**其实，Spring 的根本目的是为了保证在没有循环依赖的情况下，在引入了 AOP 之后，Bean 的生命周期设计不会得到破坏，即代理对象应该在 bean 初始化完成之后才生成，而不应该在 bean 实例化之后就生成。**

我们还是用上面存在循环依赖的 A 和 B 来举例，我们假设只使用二级缓存，并且二级缓存存放的是代理对象，那么整个流程大概就像下图这样：

![使用二级缓存时的流程](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202104211444/2021/04/21/P3q.png)

显然不管有没有循环依赖，在使用二级缓存时，代理对象总是在 bean 完全初始化之前生成。为了解决这个问题，Spring 又设计了一个 ObjectFactory 接口，这个接口是一个函数式接口，只有一个 getObject 方法用来获取实例对象。接下来我们只要在 bean 实例化之后，将它传入一个 ObjectFactory 的实现中，在这个实现中完成 AOP 的织入，最后将这个 ObjectFactory 放入缓存中即可。这样只有在其他依赖此 bean 的实例执行属性填充时，才会从三级缓存中拿到对应的 ObjectFactory，然后通过 getObject 方法创建对应的代理对象。但是，如果每个依赖此 bean 的实例都需要通过 getObject 方法来持有一个代理对象，那么显然它们持有的并不是同一个代理对象。因此我们需要再添加一个缓存容器，然后将 ObjectFactory 存放在三级缓存中，在其他实例获取三级缓存时，将直接调用获取到的 ObjectFactory 的 getObject 方法，并将结果存入二级缓存，同时删除对应的三级缓存，这样下次就可以直接从二级缓存获取代理对象了。

这就是 Spring 三级缓存的由来。在使用了三级缓存之后，如果不存在循环依赖，那么代理对象的创建会在 bean 完全初始化之后才会进行；如果存在循环依赖，整个流程大概就变成下图这样了：

![使用三级缓存时的流程](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202104211444/2021/04/21/9vo.png)

> 在 Spring 中，AOP 的功能通过 Bean 后置处理器来完成的，更确切的说，是由 `@EnableAspectJAutoProxy` 注解导入的 `AnnotationAwareAspectJAutoProxyCreator` 来完成的。在它的 postProcessAfterInitialization 方法中，同样实现了根据已有 bean 实例创建代理对象的逻辑，这也是为什么说如果不存在循环依赖，那么代理对象的创建会在 bean 完全初始化之后才会进行的原因。

# IoC 容器的依赖注入
IoC 容器的依赖注入过程是在用户第一次向容器索要 Bean 时触发的，当然也有例外，那些设置 Bean 的 lazy-init 属性为 false（默认就是 false）的 BeanDefinition 会在容器初始化时就完成预实例化。这个预实例化实际上也是一个完成依赖注入的过程，只不过它是在容器初始化的过程中完成的。

```java
/**
*  org.springframework.context.support.AbstractApplicationContext
*/
public Object getBean(String name) throws BeansException {
    assertBeanFactoryActive();
    return getBeanFactory().getBean(name);
}

/**
*  org.springframework.beans.factory.support.AbstractBeanFactory
*/
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(
    String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
    throws BeansException {
    // 转换 bean 的名称
    // 如果是别名，转换成标准名。如果首字符是 &，则会去掉所有的 &
    String beanName = transformedBeanName(name);
    Object bean;
    // 尝试从缓存中获取单例 bean
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
        // 判断当前实例是否正在创建中
        if (isSingletonCurrentlyInCreation(beanName)) {
          // 省略此处代码。这里会输出 trace 级别的日志信息，主要提醒用户该实例还没有完全初始化
        }
        else {
          logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
        }
      }
      // 如果是 FactoryBean，则会根据用户传入的 name 来决定是返回 FactoryBean 实例
      // 还是通过 FactoryBean 的 getObject 方法返回实例。如果不是 FactoryBean 则会原样返回
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    else {
      // 如果该原型实例正在创建中，则直接抛出异常
      if (isPrototypeCurrentlyInCreation(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
      }
      // 获取父容器
      BeanFactory parentBeanFactory = getParentBeanFactory();
      // 如果有父容器，并且当前容器中也没有 BeanDefinition，那么就去父容器查找
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
        String nameToLookup = originalBeanName(name);
        if (parentBeanFactory instanceof AbstractBeanFactory) {
          return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
              nameToLookup, requiredType, args, typeCheckOnly);
        }
        else if (args != null) {
          return (T) parentBeanFactory.getBean(nameToLookup, args);
        }
        else if (requiredType != null) {
          return parentBeanFactory.getBean(nameToLookup, requiredType);
        }
        else {
          return (T) parentBeanFactory.getBean(nameToLookup);
        }
      }
      // 这个标记的作用是控制是否刷新 mergedBeanDefinitions 缓存
      if (!typeCheckOnly) {
        // 标记当前 bean 为正在创建的过程中，同时清空 mergedBeanDefinitions 缓存，以防 BeanDefinition 的
        // 元数据发生变化后还读到旧的缓存数据
        markBeanAsCreated(beanName);
      }
      try {
        // 合并 BeanDefinition
        RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        checkMergedBeanDefinition(mbd, beanName, args);
        // 确保依赖的 bean 被初始化
        String[] dependsOn = mbd.getDependsOn();
        if (dependsOn != null) {
          for (String dep : dependsOn) {
            // 根据暂存的依赖关系检测是否存在循环依赖
            if (isDependent(beanName, dep)) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
            }
            // 将依赖关系暂存
            registerDependentBean(dep, beanName);
            try {
              // 获取依赖 bean
              getBean(dep);
            }
            catch (NoSuchBeanDefinitionException ex) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
            }
          }
        }
        // 创建单例 bean
        if (mbd.isSingleton()) {
          sharedInstance = getSingleton(beanName, () -> {
            try {
              return createBean(beanName, mbd, args);
            }
            catch (BeansException ex) {
              // Explicitly remove instance from singleton cache: It might have been put there
              // eagerly by the creation process, to allow for circular reference resolution.
              // Also remove any beans that received a temporary reference to the bean.
              destroySingleton(beanName);
              throw ex;
            }
          });
          bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
        // 创建原型 bean
        else if (mbd.isPrototype()) {
          Object prototypeInstance = null;
          try {
            // 在创建之前先做个标记，表示该 bean 正在创建中
            // 标记存储在一个名为 prototypesCurrentlyInCreation 的 ThreadLocal 中
            beforePrototypeCreation(beanName);
            // 创建 bean
            prototypeInstance = createBean(beanName, mbd, args);
          }
          finally {
            // 清除标记
            afterPrototypeCreation(beanName);
          }
          bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        }
        else {
          // 此处省略其他 scope 的处理
        }
      }
      catch (BeansException ex) {
        cleanupAfterBeanCreationFailure(beanName);
        throw ex;
      }
    }
    // 检查类型，如果类型不一致会尝试进行转换
    if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
        T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
        if (convertedBean == null) {
          throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
        return convertedBean;
      }
      catch (TypeMismatchException ex) {
        if (logger.isTraceEnabled()) {
          logger.trace("Failed to convert bean '" + name + "' to required type '" +
              ClassUtils.getQualifiedName(requiredType) + "'", ex);
        }
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
    }
    return (T) bean;
}
```

获取 Bean 的方法最终调用的是 doGetBean 方法。在该方法中，首先尝试从三个级别的缓存中获取单例 bean。

```java
/**
*  org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
*/
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 先尝试从一级缓存获取
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      // 再尝试从二级缓存获取
      singletonObject = this.earlySingletonObjects.get(beanName);
      // 如果允许提前暴露单例 bean
      if (singletonObject == null && allowEarlyReference) {
        synchronized (this.singletonObjects) {
          // Consistent creation of early reference within full singleton lock
          singletonObject = this.singletonObjects.get(beanName);
          if (singletonObject == null) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            // 尝试从三级缓存中获取
            if (singletonObject == null) {
              ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
              if (singletonFactory != null) {
                singletonObject = singletonFactory.getObject();
                // 获取到之后放入二级缓存同时删除三级缓存
                this.earlySingletonObjects.put(beanName, singletonObject);
                this.singletonFactories.remove(beanName);
              }
            }
          }
        }
      }
    }
    return singletonObject;
}
```

如果缓存为空，那么会有三种可能：实例在父容器中，从未创建和初始化过该 bean，这是一个原型 bean。所以接下来会在父容器存在，并且当前容器中没有对应的 BeanDefinition 时，尝试从父容器获取 bean 的实例。如果父容器不存在，或者当前容器中存在对应的 BeanDefinition 时，继续向下执行获取 bean 的过程。

在这个过程中，如果 BeanDefinition 存在继承关系，那么还需要使用 getMergedLocalBeanDefinition 方法合并它们。

```java
/**
*  org.springframework.beans.factory.support.AbstractBeanFactory
*/
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
    // Quick check on the concurrent map first, with minimal locking.
    RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
    if (mbd != null) {
      return mbd;
    }
    return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}

protected RootBeanDefinition getMergedBeanDefinition(String beanName, BeanDefinition bd)
    throws BeanDefinitionStoreException {
    return getMergedBeanDefinition(beanName, bd, null);
}

protected RootBeanDefinition getMergedBeanDefinition(
    String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
    throws BeanDefinitionStoreException {
    synchronized (this.mergedBeanDefinitions) {
      RootBeanDefinition mbd = null;
      // 先尝试从缓存中获取
      if (containingBd == null) {
        mbd = this.mergedBeanDefinitions.get(beanName);
      }
      if (mbd == null) {
        // 如果当前的 BeanDefinition 没有父级的 BeanDefinition
        if (bd.getParentName() == null) {
          // 如果它是 RootBeanDefinition 类型的，则再克隆一个出来
          if (bd instanceof RootBeanDefinition) {
            mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
          }
          // 如果它不是 RootBeanDefinition 类型的，那么就使用 BeanDefinition 构造一个
          else {
            mbd = new RootBeanDefinition(bd);
          }
        }
        else {
          // 能走到这里代表当前 BeanDefinition 是一个 Child，因此需要合并父 BeanDefinition
          BeanDefinition pbd;
          try {
            // 如果当前 bean 名称与父 bean 名称不同
            // 那么使用父 bean 的名称递归调用 getMergedBeanDefinition 方法
            // 这样可以确保具有继承关系的 BeanDefinition 都被合并
            String parentBeanName = transformedBeanName(bd.getParentName());
            if (!beanName.equals(parentBeanName)) {
              pbd = getMergedBeanDefinition(parentBeanName);
            }
            else {
              // 如果名称相同，则直接通过父容器调用 getMergedBeanDefinition 方法
              BeanFactory parent = getParentBeanFactory();
              if (parent instanceof ConfigurableBeanFactory) {
                pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
              }
              else {
                throw new NoSuchBeanDefinitionException(parentBeanName,
                    "Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
                    "': cannot be resolved without a ConfigurableBeanFactory parent");
              }
            }
          }
          catch (NoSuchBeanDefinitionException ex) {
            throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
                "Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
          }
          // 先使用父级 BeanDefinition 构建一个 RootBeanDefinition
          mbd = new RootBeanDefinition(pbd);
          // 然后将父子 BeanDefinition 合并
          mbd.overrideFrom(bd);
        }
        // Set default singleton scope, if not configured before.
        if (!StringUtils.hasLength(mbd.getScope())) {
          mbd.setScope(SCOPE_SINGLETON);
        }
        if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
          mbd.setScope(containingBd.getScope());
        }
        // 放入缓存
        if (containingBd == null && isCacheBeanMetadata()) {
          this.mergedBeanDefinitions.put(beanName, mbd);
        }
      }
      return mbd;
    }
}
```

接着还要处理 BeanDefinition 的 dependsOn 属性，这个依赖可以理解为只有 dependsOn 属性中指定的 bean 先完成初始化之后，该 bean 才能进行初始化。这就意味着，该 bean 并不需要持有依赖对象，如果持有的话直接使用 ref 即可。在这些前期工作完成以后，接下来会通过 `getSingleton(beanName, singletonFactory)` 方法来获取 bean 的实例。

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
      // 先尝试从一级缓存获取
      Object singletonObject = this.singletonObjects.get(beanName);
      if (singletonObject == null) {
        // 在销毁该容器的单例 bean 时，不允许创建单例 Bean
        if (this.singletonsCurrentlyInDestruction) {
          throw new BeanCreationNotAllowedException(beanName,
              "Singleton bean creation not allowed while singletons of this factory are in destruction " +
              "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
        }
        if (logger.isDebugEnabled()) {
          logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
        }
        // 在创建单例 bean 之前做一些检查和标记，比如设置 singletonsCurrentlyInCreation
        beforeSingletonCreation(beanName);
        boolean newSingleton = false;
        boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
        if (recordSuppressedExceptions) {
          this.suppressedExceptions = new LinkedHashSet<>();
        }
        try {
          // 通过 ObjectFactory 获取单例 bean，注意这个 bean 是一个完成初始化（依赖注入）的 bean
          singletonObject = singletonFactory.getObject();
          newSingleton = true;
        }
        catch (IllegalStateException ex) {
          // Has the singleton object implicitly appeared in the meantime ->
          // if yes, proceed with it since the exception indicates that state.
          singletonObject = this.singletonObjects.get(beanName);
          if (singletonObject == null) {
            throw ex;
          }
        }
        catch (BeanCreationException ex) {
          if (recordSuppressedExceptions) {
            for (Exception suppressedException : this.suppressedExceptions) {
              ex.addRelatedCause(suppressedException);
            }
          }
          throw ex;
        }
        finally {
          if (recordSuppressedExceptions) {
            this.suppressedExceptions = null;
          }
          // 在创建单例 bean 之后做一些检查和标记
          afterSingletonCreation(beanName);
        }
        // 如果是新建的 bean，将它放入一级缓存，同时删除对应的二级和三级缓存
        if (newSingleton) {
          addSingleton(beanName, singletonObject);
        }
      }
      return singletonObject;
    }
}
```

在该方法中，我们可以看到主要的一行代码就是调用 ObjectFactory 的 getObject 方法，这个方法实际上调用的是 createBean 方法来创建并初始化一个 bean 的实例。由于这个实例是一个完成依赖注入之后的实例，因此最后还会将它放入一级缓存中。

```java
/**
*  org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*/
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    RootBeanDefinition mbdToUse = mbd;
    // 确保需要创建 bean 实例的类可以被实例化
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
    }
    // Prepare method overrides.
    try {
      mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
          beanName, "Validation of method overrides failed", ex);
    }
    try {
      // 在 bean 实例化之前完成一些用户自定义的操作
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
        return bean;
      }
    } catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
          "BeanPostProcessor before instantiation of bean failed", ex);
    }
    try {
      // 创建 bean
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
        logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
    } catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      throw ex;
    } catch (Throwable ex) {
      throw new BeanCreationException(
          mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}
```

在 createBean 方法中，首先会检查需要创建 bean 实例的类是否可以被实例化。如果容器配置了 BeanPostProcessor，那么在实例化 bean 之前还会调用 Bean 后置处理器的 postProcessBeforeInstantiation 方法完成一些用户自定义的操作。该方法返回的 bean 对象可以是代替目标 bean 的代理对象，这样目标 bean 就不会执行默认的实例化操作，唯一会执行的进一步处理就是调用 Bean 后置处理器的 postProcessAfterInitialization 方法来完成 bean 的初始化操作，然后直接返回初始化完成的 bean。在方法的最后，doCreateBean 才是真正创建并初始化 bean 实例的方法。

```java
/**
*  org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*/
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    // 这个 BeanWrapper 用来包装实例化的 bean
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
      // 创建 bean 实例
      instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
    }
    // 此处将允许 MergedBeanDefinitionPostProcessor 对 BeanDefinition 进行处理
    synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
        try {
          applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
        }
        catch (Throwable ex) {
          throw new BeanCreationException(mbd.getResourceDescription(), beanName,
              "Post-processing of merged bean definition failed", ex);
        }
        mbd.postProcessed = true;
      }
    }
    // 如果容器允许单例 bean 出现循环依赖，同时此单例 bean 还在创建中，那么提早将这个单例 bean 放入缓存（三级缓存）
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
        logger.trace("Eagerly caching bean '" + beanName +
            "' to allow for resolving potential circular references");
      }
      // 将单例 bean 放入三级缓存 singletonFactories 中
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    Object exposedObject = bean;
    try {
      // bean 的依赖注入
      populateBean(beanName, mbd, instanceWrapper);
      // 执行一些初始化操作
      exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
      // 省略部分代码
    }
    if (earlySingletonExposure) {
      // 尝试从一级缓存和二级缓存中获取实例
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
        if (exposedObject == bean) {
          exposedObject = earlySingletonReference;
        }
        else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
          String[] dependentBeans = getDependentBeans(beanName);
          Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
          for (String dependentBean : dependentBeans) {
            if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
              actualDependentBeans.add(dependentBean);
            }
          }
          if (!actualDependentBeans.isEmpty()) {
            throw new BeanCurrentlyInCreationException(beanName,
                "Bean with name '" + beanName + "' has been injected into other beans [" +
                StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                "] in its raw version as part of a circular reference, but has eventually been " +
                "wrapped. This means that said other beans do not use the final version of the " +
                "bean. This is often the result of over-eager type matching - consider using " +
                "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
          }
        }
      }
    }
    // Register bean as disposable.
    try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
          mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

在 doCreateBean 方法中，通过 createBeanInstance 方法来创建 bean 的实例。如果 Spring 容器允许单例 bean 出现循环依赖，那么这个未初始化的 bean 实例会被放入三级缓存中。接下来通过 populateBean 方法对 bean 实例执行属性值的依赖注入，然后使用 initializeBean 方法执行一些初始化的操作。最后通过 registerDisposableBeanIfNecessary 方法注册一些销毁 bean 的回调方法，比如一些 DestructionAwareBeanPostProcessor 接口，DisposableBean 接口，以及自定义的 `destroy-method` 方法。将未初始化的 bean 实例放入三级缓存这部分代码最简单，我们先来看它。

```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));

protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
      // 一级缓存中没有
      if (!this.singletonObjects.containsKey(beanName)) {
        // 放入三级缓存
        this.singletonFactories.put(beanName, singletonFactory);
        // 删除二级缓存中对应的数据
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
      }
    }
}

protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
          SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
          // 调用 getEarlyBeanReference 方法
          exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
        }
      }
    }
    return exposedObject;
}
```

其实我想说的是 getEarlyBeanReference 方法，这个方法在容器没有配置 BeanPostProcessor 时只会返回之前未完成初始化的 bean 实例，而如果容器设置了这类 Bean 后置处理器，这里就会调用它们的 getEarlyBeanReference 方法来获得用户特殊处理过的 bean 实例。这个扩展点很多时候是被用来返回经过 AOP 处理的代理 bean 对象。

我们回到刚才的 doCreateBean 方法，继续查看 createBeanInstance 部分的代码。

```java
/**
*  org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*/
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // Make sure bean class is actually resolved at this point. 
    // 确保需要创建 bean 实例的类可以被实例化
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
          "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }
    // 如果存在 Supplier 回调，则使用给定的回调方法进行实例化
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
      return obtainFromSupplier(instanceSupplier, beanName);
    }
    // 这里使用 FactoryBean 的工厂方法对 bean 进行实例化，需要注意的是该工厂方法可以是一个静态方法
    if (mbd.getFactoryMethodName() != null) {
      return instantiateUsingFactoryMethod(beanName, mbd, args);
    }
    // 由于 Spring 需要根据参数确认到底使用哪个构造函数，该过程比较耗时，所以采用了缓存机制
    // 将解析过的数据放在 BeanDefinition 中，下次创建相同 bean 时能够提高效率
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
      synchronized (mbd.constructorArgumentLock) {
        // resolvedConstructorOrFactoryMethod 用于缓存已解析的构造函数或工厂方法
        // 如果它不为空，说明缓存中已经有解析好的构造函数或工厂方法
        if (mbd.resolvedConstructorOrFactoryMethod != null) {
          resolved = true;
          // constructorArgumentsResolved 用于标记构造函数的参数是否已经解析完毕
          autowireNecessary = mbd.constructorArgumentsResolved;
        }
      }
    }
    // 构造函数或工厂方法已经解析过（知道该用哪个）
    if (resolved) {
      // 如果参数也已经解析过了，那么可以直接使用构造函数进行实例化
      if (autowireNecessary) {
        return autowireConstructor(beanName, mbd, null, null);
      }
      else {
        // 使用默认的无参构造函数进行实例化
        return instantiateBean(beanName, mbd);
      }
    }
    // 这里尝试从 SmartInstantiationAwareBeanPostProcessor 中获取候选构造函数
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
      return autowireConstructor(beanName, mbd, ctors, args);
    }
    // 使用首选构造函数进行实例化。比如 Kotlin 中的主要构造器（Primary Constructor）
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
      return autowireConstructor(beanName, mbd, ctors, null);
    }
    // 无需特殊的处理，直接使用默认的无参构造函数进行实例化
    return instantiateBean(beanName, mbd);
}
```

在 createBeanInstance 方法中，大体上有四种实例化 bean 的方式。obtainFromSupplier 方法会从 BeanDefinition 中获取一个名为 instanceSupplier 的 Supplier，然后通过这个 Supplier 获取 bean 的实例。instantiateUsingFactoryMethod 使用 FactoryBean 的工厂方法来创建 bean 的实例，需要注意的是该工厂方法可以是一个静态方法。autowireConstructor 方法则是通过构造方法完成 bean 的实例化，不过由于构造函数和构造参数的不确定性，这个方法在没有提供候选的构造函数时需要花大量的精力来确定构造函数和构造参数（可以通过 BeanPostProcessor 提供候选构造器）。最后的 instantiateBean 方法则是使用默认构造函数进行实例化。

我们回到上面的 doCreateBean 方法，继续查看 populateBean 部分的代码。

```java
/**
*  org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*/
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // 空对象无法注入
    if (bw == null) {
      if (mbd.hasPropertyValues()) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
        // Skip property population phase for null instance.
        return;
      }
    }
    // 这里会调用 InstantiationAwareBeanPostProcessor 的 postProcessAfterInstantiation 方法
    // 执行 bean 实例化后的操作
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
          InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
            return;
          }
        }
      }
    }
    // 从 BeanDefinition 获取 property 值
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
    // 这里是对 autowire 的处理
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
      // 根据名称执行自动注入
      if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, newPvs);
      }
      // 根据类型执行自动注入
      if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
    }
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
      if (pvs == null) {
        pvs = mbd.getPropertyValues();
      }
      // 这里会调用 InstantiationAwareBeanPostProcessor 的 postProcessProperties 方法
      // 在 bean 注入属性值之前，对属性值进行处理
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
          InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
          if (pvsToUse == null) {
            if (filteredPds == null) {
              filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            }
            pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
              return;
            }
          }
          pvs = pvsToUse;
        }
      }
    }
    if (needsDepCheck) {
      if (filteredPds == null) {
        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }
      checkDependencies(beanName, mbd, filteredPds, pvs);
    }
    // 对属性执行注入
    if (pvs != null) {
      applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

在该方法中，如果容器配置了 Bean 后置处理器，那么就会执行它们的 postProcessAfterInstantiation 方法来完成用户自定义的一些 bean 实例化后的操作。如果 BeanDefinition 配置了自动注入的模式，那么还会选择对应的模式对该 bean 的属性值执行注入。接下来还会执行 Bean 后置处理器的 postProcessProperties 方法，在 bean 注入属性值之前，对属性值进行一些自定义的处理。最终会调用 applyPropertyValues 方法，对 bean 的属性值执行注入。

```java
/**
*  org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*/
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    if (pvs.isEmpty()) {
      return;
    }
    if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
      ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
    }
    MutablePropertyValues mpvs = null;
    List<PropertyValue> original;
    if (pvs instanceof MutablePropertyValues) {
      mpvs = (MutablePropertyValues) pvs;
      // 已经过转换，则直接设置属性值
      if (mpvs.isConverted()) {
        // Shortcut: use the pre-converted values as-is.
        try {
          bw.setPropertyValues(mpvs);
          return;
        }
        catch (BeansException ex) {
          throw new BeanCreationException(
              mbd.getResourceDescription(), beanName, "Error setting property values", ex);
        }
      }
      original = mpvs.getPropertyValueList();
    }
    else {
      original = Arrays.asList(pvs.getPropertyValues());
    }
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
      converter = bw;
    }
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);
    // 创建一个属性值的深拷贝列表，最终会使用该列表对 bean 执行属性值注入
    List<PropertyValue> deepCopy = new ArrayList<>(original.size());
    boolean resolveNecessary = false;
    for (PropertyValue pv : original) {
      // 属性值已经过转换，则直接添加到列表中
      if (pv.isConverted()) {
        deepCopy.add(pv);
      }
      else {
        // 执行转换
        String propertyName = pv.getName();
        Object originalValue = pv.getValue();
        // 如果有必要，使用 BeanDefinitionValueResolver 转换属性值
        Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
        Object convertedValue = resolvedValue;
        boolean convertible = bw.isWritableProperty(propertyName) &&
            !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
        if (convertible) {
          convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
        }
        // 将转换后的值放入 PropertyValue 进行缓存
        if (resolvedValue == originalValue) {
          if (convertible) {
            // 将转换后的属性值缓存起来
            pv.setConvertedValue(convertedValue);
          }
          deepCopy.add(pv);
        }
        else if (convertible && originalValue instanceof TypedStringValue &&
            !((TypedStringValue) originalValue).isDynamic() &&
            !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
          pv.setConvertedValue(convertedValue);
          deepCopy.add(pv);
        }
        else {
          resolveNecessary = true;
          deepCopy.add(new PropertyValue(pv, convertedValue));
        }
      }
    }
    if (mpvs != null && !resolveNecessary) {
      mpvs.setConverted();
    }
    // 设置处理后的属性值
    try {
      bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    }
    catch (BeansException ex) {
      throw new BeanCreationException(
          mbd.getResourceDescription(), beanName, "Error setting property values", ex);
    }
}
```

在该方法中，主要工作是对属性值进行处理，这项工作由 BeanDefinitionValueResolver 的 resolveValueIfNecessary 方法完成。为此还创建了一个属性值的深拷贝列表，所有已经处理过的属性值都会被放入该列表中。当所有的属性值都处理完毕之后，使用 BeanWrapper 的 setPropertyValues 方法，将这个深拷贝列表传入，执行属性值注入的过程。

```java
/**
*  org.springframework.beans.factory.support.BeanDefinitionValueResolver
*/
public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
    // 属性值可以能是另一个 bean 的引用
    // 这个 RuntimeBeanReference 是在载入 BeanDefinition 时根据配置生成的
    if (value instanceof RuntimeBeanReference) {
      RuntimeBeanReference ref = (RuntimeBeanReference) value;
      return resolveReference(argName, ref);
    }
    // 省略部分代码
    else if (value instanceof NullBean) {
      return null;
    }
    else {
      return evaluate(value);
    }
}

private Object resolveReference(Object argName, RuntimeBeanReference ref) {
    try {
      Object bean;
      String refName = ref.getBeanName();
      refName = String.valueOf(doEvaluate(refName));
      if (ref.isToParent()) {
        if (this.beanFactory.getParentBeanFactory() == null) {
          throw new BeanCreationException(
              this.beanDefinition.getResourceDescription(), this.beanName,
              "Can't resolve reference to bean '" + refName +
                  "' in parent factory: no parent factory available");
        }
        // 从父容器获取 bean
        bean = this.beanFactory.getParentBeanFactory().getBean(refName);
      }
      else {
        // 从当前容器获取 bean
        bean = this.beanFactory.getBean(refName);
        this.beanFactory.registerDependentBean(refName, this.beanName);
      }
      if (bean instanceof NullBean) {
        bean = null;
      }
      return bean;
    }
    catch (BeansException ex) {
      // 省略部分代码
    }
}
```

我们回到上面的 doCreateBean 方法，继续查看 initializeBean 部分的代码。

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
        invokeAwareMethods(beanName, bean);
        return null;
      }, getAccessControlContext());
    }
    else {
      invokeAwareMethods(beanName, bean);
    }
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    try {
      invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
      throw new BeanCreationException(
          (mbd != null ? mbd.getResourceDescription() : null),
          beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

在该方法中，invokeAwareMethods 方法用来向 Aware 类型的接口注入它们需要的参数，比如 beanName、beanFactory 等。如果容器配置了 Bean 后置处理器，那么还会调用它们的 postProcessBeforeInitialization 方法，在 bean 完全初始化之前执行一些用户自定义的操作。如果当前 bean 是一个 `InitializingBean`，那么还会调用它的 afterPropertiesSet 方法完成一些自定义的配置验证和最终初始化工作。接下来会调用 bean 的 `init-method` 方法。当然，最终还会调用 Bean 后置处理器的 postProcessAfterInitialization 方法来，在 bean 完全初始化之后完成一些用户自定义的操作。

经过上面的分析，我们已经了解了 doCreateBean 方法的细节，接下来我们回到 doGetBean 方法，来看看 Spring 是如何处理最终返回给用户的 bean 实例的。

```java
/**
*  org.springframework.beans.factory.support.AbstractBeanFactory
*/
protected Object getObjectForBeanInstance(
    Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
    // 检查用户传入的 name 值，如果首字符是 &，而从容器中获取到的却不是 FactoryBean，则直接报错
    if (BeanFactoryUtils.isFactoryDereference(name)) {
      if (beanInstance instanceof NullBean) {
        return beanInstance;
      }
      if (!(beanInstance instanceof FactoryBean)) {
        throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
      }
    }
    // 如果用户想要获取 FactoryBean 本身，或者从容器获取到的不是 FactoryBean，直接返回该 bean
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
      return beanInstance;
    }
    // 能走到这里说明用户想要获取的不是 FactoryBean，而是通过 FactoryBean 得到的 bean 实例
    Object object = null;
    // 先尝试从缓存中获取
    if (mbd == null) {
      object = getCachedObjectForFactoryBean(beanName);
    }
    if (object == null) {
      FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
      // Caches object obtained from FactoryBean if it is a singleton.
      if (mbd == null && containsBeanDefinition(beanName)) {
        mbd = getMergedLocalBeanDefinition(beanName);
      }
      boolean synthetic = (mbd != null && mbd.isSynthetic());
      // 通过 FactoryBean 获取 bean 实例
      object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}

protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    if (factory.isSingleton() && containsSingleton(beanName)) {
      synchronized (getSingletonMutex()) {
        // 先尝试从缓存获取
        Object object = this.factoryBeanObjectCache.get(beanName);
        if (object == null) {
          // 调用 FactoryBean 的 getObject 方法获取 bean 实例
          object = doGetObjectFromFactoryBean(factory, beanName);
          // Only post-process and store if not put there already during getObject() call above
          // (e.g. because of circular reference processing triggered by custom getBean calls)
          Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
          if (alreadyThere != null) {
            object = alreadyThere;
          }
          else {
            if (shouldPostProcess) {
              if (isSingletonCurrentlyInCreation(beanName)) {
                // Temporarily return non-post-processed object, not storing it yet..
                return object;
              }
              beforeSingletonCreation(beanName);
              try {
                // 后置处理
                object = postProcessObjectFromFactoryBean(object, beanName);
              }
              catch (Throwable ex) {
                throw new BeanCreationException(beanName,
                    "Post-processing of FactoryBean's singleton object failed", ex);
              }
              finally {
                afterSingletonCreation(beanName);
              }
            }
            if (containsSingleton(beanName)) {
              this.factoryBeanObjectCache.put(beanName, object);
            }
          }
        }
        return object;
      }
    }
    else {
      Object object = doGetObjectFromFactoryBean(factory, beanName);
      if (shouldPostProcess) {
        try {
          object = postProcessObjectFromFactoryBean(object, beanName);
        }
        catch (Throwable ex) {
          throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
        }
      }
      return object;
    }
}
```

简单说来，如果用户想要获取的 bean 是一个 FactoryBean，那么根据用户传入的 bean 名称（首字符是否为 `&`），选择返回 FactoryBean 对象还是返回通过调用它的 getObject 方法返回的对象。而如果用户想要获取的 bean 就是一个普通的 bean，那么直接返回。

# Spring Bean 生命周期
在 Spring bean 的完整生命周期中，Spring 提供了一系列的扩展方法方便我们调用，可以将这些方法大体划分为：bean 级别和容器级别。

bean 级别的方法包括 bean 本身的方法，比如通过配置文件定义的 `init-method` 方法和 `destroy-method` 方法，bean 实现 `InitializingBean`、`BeanNameAware`、`BeanFactoryAware`、`DisposableBean` 等接口的方法。

容器级别包括 bean 后置处理器 `BeanPostProcessor` 和 `InstantiationAwareBeanPostProcessor` 接口的方法，bean 工厂后置处理器 `BeanFactoryPostProcessor` 接口的方法。

![Spring bean 生命周期 ](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/05/31/dQz.png)  

# 参考

> 《Spring 技术内幕：深入解析 Spring 架构与计原理 (第 2 版)》

