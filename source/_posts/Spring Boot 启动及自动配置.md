---
title: Spring Boot 启动及自动配置
date: 2017/7/22 10:24:0
tags: [Spring Boot]
categories: [Spring Boot]
---

> 本文分析使用的 Spring Boot 版本为 Spring Boot 2.1.18.RELEASE

一个简单的 Spring Boot 启动类：  

<!--more-->  

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
		
`@SpringBootApplication` 主要包含 `@SpringBootConfiguration`、`@EnableAutoConfiguration` 和 `@ComponentScan` 三个注解，能够实现自动配置的是 `@EnableAutoConfiguration` 注解。  

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
  String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
  Class<?>[] exclude() default {};
  String[] excludeName() default {};
}
```

Spring 框架提供了许多以 @Enable 开头的注解，比如 `@EnableScheduling`、`@EnableCaching`、`@EnableMBeanExport` 等，`@EnableAutoConfiguration` 的理念和做事方式其实一脉相承，简单概括一下就是，借助 `@Import` 的支持，收集和注册特定场景相关的 bean。`@EnableAutoConfiguration` 就是通过 `@Import` 的帮助，将所有符合自动配置条件的 bean 加载到 IoC 容器中的。

Spring 对于 `@Import` 注解的处理是在 ConfigurationClassPostProcessor 类的 postProcessBeanFactory 方法中。

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  int factoryId = System.identityHashCode(beanFactory);
  if (this.factoriesPostProcessed.contains(factoryId)) {
    throw new IllegalStateException(
        "postProcessBeanFactory already called on this post-processor against " + beanFactory);
  }
  this.factoriesPostProcessed.add(factoryId);
  if (!this.registriesPostProcessed.contains(factoryId)) {
    // 处理所有的配置类 BeanDefinition
    processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
  }
  enhanceConfigurationClasses(beanFactory);
  beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}
```

所有的 BeanDefinition 在容器初始化阶段就注册了，但是有些 BeanDefinition 可能是配置类，因此需要解析这些配置类（配置类可能包含其他的 BeanDefinition）。

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
  List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
  // 从 registry 获取所有的 BeanDefinition
  String[] candidateNames = registry.getBeanDefinitionNames();
  for (String beanName : candidateNames) {
    BeanDefinition beanDef = registry.getBeanDefinition(beanName);
    // 检查是否是配置类
    if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
        ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
      if (logger.isDebugEnabled()) {
        logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
      }
    }
    // 判断是否是一个配置类，并为 BeanDefinition 设置属性为 lite 或者 full，方便后续的使用
    // 如果使用了 @Configuration 注解则设置属性为 full
    // 如果使用了 @Bean、@Component、@ComponentScan、@Import、@ImportResource 注解，则设置属性为 lite
    // 如果使用了 @Order 注解，则为 BeanDefinition 设置 order 属性值
    else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
      // 将该配置类 Holder 添加到 configCandidates
      configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
    }
  }
  // 如果没有发现任何配置类，则直接返回
  if (configCandidates.isEmpty()) {
    return;
  }
  // 根据 order 排序
  configCandidates.sort((bd1, bd2) -> {
    int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
    int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
    return Integer.compare(i1, i2);
  });
  // Detect any custom bean name generation strategy supplied through the enclosing application context
  SingletonBeanRegistry sbr = null;
  if (registry instanceof SingletonBeanRegistry) {
    sbr = (SingletonBeanRegistry) registry;
    if (!this.localBeanNameGeneratorSet) {
      BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
      if (generator != null) {
        this.componentScanBeanNameGenerator = generator;
        this.importBeanNameGenerator = generator;
      }
    }
  }
  if (this.environment == null) {
    this.environment = new StandardEnvironment();
  }
  // 解析每个配置类
  ConfigurationClassParser parser = new ConfigurationClassParser(
      this.metadataReaderFactory, this.problemReporter, this.environment,
      this.resourceLoader, this.componentScanBeanNameGenerator, registry);
  // 将所有的配置类放入候选集合中
  Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
  // 已经解析过的配置类
  Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
  do {
    // 解析配置类
    parser.parse(candidates);
    // 验证
    parser.validate();
    // 获取所有解析过的配置类
    Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
    configClasses.removeAll(alreadyParsed);
    // 创建 BeanDefinitionReader
    if (this.reader == null) {
      this.reader = new ConfigurationClassBeanDefinitionReader(
          registry, this.sourceExtractor, this.resourceLoader, this.environment,
          this.importBeanNameGenerator, parser.getImportRegistry());
    }
    // 将完全填充好的 ConfigurationClass 实例转化成 BeanDefinition 注册到容器
    this.reader.loadBeanDefinitions(configClasses);
    alreadyParsed.addAll(configClasses);
    // 省略部分代码
  }
  while (!candidates.isEmpty());
  // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
  if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
    sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
  }
  if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
    ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
  }
}
```

该方法会筛选出容器中所有的配置类 BeanDefinition，然后解析它们。最终所有的 BeanDefinition 都能被注册到容器中。

```java
/**
*   org.springframework.context.annotation.ConfigurationClassParser
*/
public void parse(Set<BeanDefinitionHolder> configCandidates) {
  for (BeanDefinitionHolder holder : configCandidates) {
    BeanDefinition bd = holder.getBeanDefinition();
    try {
      if (bd instanceof AnnotatedBeanDefinition) {
        parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
      }
      else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
        parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
      }
      else {
        parse(bd.getBeanClassName(), holder.getBeanName());
      }
    }
    catch (BeanDefinitionStoreException ex) {
      throw ex;
    }
    catch (Throwable ex) {
      throw new BeanDefinitionStoreException(
          "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
    }
  }
  this.deferredImportSelectorHandler.process();
}

protected final void parse(@Nullable String className, String beanName) throws IOException {
  Assert.notNull(className, "No bean class name for configuration class bean definition");
  MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
  processConfigurationClass(new ConfigurationClass(reader, beanName));
}

protected final void parse(Class<?> clazz, String beanName) throws IOException {
  processConfigurationClass(new ConfigurationClass(clazz, beanName));
}

protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
  processConfigurationClass(new ConfigurationClass(metadata, beanName));
}
```

该方法主要是根据不同类型的 BeanDefinition 使用不同的解析方式。


```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
  // 根据条件注解判断是否跳过解析过程
  if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
    return;
  }
  ConfigurationClass existingClass = this.configurationClasses.get(configClass);
  if (existingClass != null) {
    if (configClass.isImported()) {
      if (existingClass.isImported()) {
        existingClass.mergeImportedBy(configClass);
      }
      // Otherwise ignore new imported config class; existing non-imported class overrides it.
      return;
    }
    else {
      // 删除旧的，在下面替换成新的
      this.configurationClasses.remove(configClass);
      this.knownSuperclasses.values().removeIf(configClass::equals);
    }
  }
  // 递归处理配置类及其超类
  SourceClass sourceClass = asSourceClass(configClass);
  do {
    sourceClass = doProcessConfigurationClass(configClass, sourceClass);
  }
  while (sourceClass != null);
  // 将处理后的配置类放入
  this.configurationClasses.put(configClass, configClass);
}
```


```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
    throws IOException {
  // 省略部分代码
  
  // Process any @Import annotations
  processImports(configClass, sourceClass, getImports(sourceClass), true);
  // 省略部分代码
}
```


```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
    Collection<SourceClass> importCandidates, boolean checkForCircularImports) {
  if (importCandidates.isEmpty()) {
    return;
  }
  if (checkForCircularImports && isChainedImportOnStack(configClass)) {
    this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
  }
  else {
    this.importStack.push(configClass);
    try {
      for (SourceClass candidate : importCandidates) {
        // 如果是 ImportSelector
        if (candidate.isAssignable(ImportSelector.class)) {
          Class<?> candidateClass = candidate.loadClass();
          ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
          ParserStrategyUtils.invokeAwareMethods(
              selector, this.environment, this.resourceLoader, this.registry);
          // 如果是 DeferredImportSelector
          if (selector instanceof DeferredImportSelector) {
            this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
          }
          // 不是的话就调用 selectImports 方法
          else {
            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
            processImports(configClass, currentSourceClass, importSourceClasses, false);
          }
        }
        // 省略部分代码
      }
    }
    catch (BeanDefinitionStoreException ex) {
      throw ex;
    }
    catch (Throwable ex) {
      // 省略部分代码
    }
    finally {
      this.importStack.pop();
    }
  }
}
```

在该方法中，如果导入的是一个 ImportSelector，那么会调用它的 selectImports 方法获取更多需要导入到容器的类。由于 AutoConfigurationImportSelector 是一个 DeferredImportSelector，这里会执行 handle 方法。

```java
/**
*   org.springframework.context.annotation.ConfigurationClassParser$DeferredImportSelectorHandler
*/
private class DeferredImportSelectorHandler {
  // deferredImportSelectors 在类创建时就会初始化
  private List<DeferredImportSelectorHolder> deferredImportSelectors = new ArrayList<>();

  public void handle(ConfigurationClass configClass, DeferredImportSelector importSelector) {
    DeferredImportSelectorHolder holder = new DeferredImportSelectorHolder(configClass, importSelector);
    if (this.deferredImportSelectors == null) {
      DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
      handler.register(holder);
      handler.processGroupImports();
    }
    else {
      // 第一次调用时会走这里
      this.deferredImportSelectors.add(holder);
    }
  }
}
```

在第一次调用该方法时，deferredImportSelectors 已经初始化，因此只将 DeferredImportSelectorHolder 放入其中。在回到 parse 方法时，则会调用 deferredImportSelectorHandler 的 process 方法。

```java
/**
*   org.springframework.context.annotation.ConfigurationClassParser$DeferredImportSelectorHandler
*/
public void process() {
  List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
  // 置为 null
  this.deferredImportSelectors = null;
  try {
    if (deferredImports != null) {
      // 创建 DeferredImportSelectorGroupingHandler
      DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
      deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
      // 对每个 DeferredImportSelectorHolder 执行 register 方法，将它们添加到对应的分组中
      deferredImports.forEach(handler::register);
      // 处理每个分组
      handler.processGroupImports();
    }
  }
  finally {
    this.deferredImportSelectors = new ArrayList<>();
  }
}
```

```java
public void register(DeferredImportSelectorHolder deferredImport) {
  // 获取 deferredImport 对应的分组
  // 目前只会返回 AutoConfigurationImportSelector$AutoConfigurationGroup.class
  Class<? extends Group> group = deferredImport.getImportSelector().getImportGroup();
  // 创建 DeferredImportSelectorGrouping，createGroup 方法将实例化这个 Group 类型
  DeferredImportSelectorGrouping grouping = this.groupings.computeIfAbsent(
      (group != null ? group : deferredImport),
      key -> new DeferredImportSelectorGrouping(createGroup(group)));
  // 将 deferredImport 也放入其中
  grouping.add(deferredImport);
  this.configurationClasses.put(deferredImport.getConfigurationClass().getMetadata(),
      deferredImport.getConfigurationClass());
}

// DeferredImportSelectorGrouping 的结构
private static class DeferredImportSelectorGrouping {
  // 分组
  private final DeferredImportSelector.Group group;
  // 分组下的 DeferredImportSelector 集合
  private final List<DeferredImportSelectorHolder> deferredImports = new ArrayList<>();
  DeferredImportSelectorGrouping(Group group) {
    this.group = group;
  }
  public void add(DeferredImportSelectorHolder deferredImport) {
    this.deferredImports.add(deferredImport);
  }
  public Iterable<Group.Entry> getImports() {
    for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
      // 先调用 Group 的 process 方法
      this.group.process(deferredImport.getConfigurationClass().getMetadata(),
          deferredImport.getImportSelector());
    }
    // 再调用 Group 的 selectImports 方法
    return this.group.selectImports();
  }
}
```

在 Spring 5 中，DeferredImportSelector 接口新增了一个内部接口 Group，这个接口的作用是对 ImportSelector 进行分组。register 方法的作用是将每个 DeferredImportSelector 存放到它对应的分组之中。在该方法中，首先调用 getImportGroup 拿到 Group 类型的类，然后通过 createGroup 方法实例化这个类，最后通过这个 Group 实例构建一个 DeferredImportSelectorGrouping 并放入 `this.groupings` 中。

```java
public void processGroupImports() {
  for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
    grouping.getImports().forEach(entry -> {
      ConfigurationClass configurationClass = this.configurationClasses.get(entry.getMetadata());
      try {
        processImports(configurationClass, asSourceClass(configurationClass),
            asSourceClasses(entry.getImportClassName()), false);
      }
      catch (BeanDefinitionStoreException ex) {
        throw ex;
      }
      catch (Throwable ex) {
        throw new BeanDefinitionStoreException(
            "Failed to process import candidates for configuration class [" +
                configurationClass.getMetadata().getClassName() + "]", ex);
      }
    });
  }
}
```

接下来调用 processGroupImports 方法。在该方法中会遍历所有的分组，每个分组分别执行对应的 process 方法和 selectImports 方法。selectImports 方法能够获取到所有经过处理的自动配置类集合，最后遍历这个集合并通过一开始我们就提到过的 processImports 方法逐个解析配置类。

```java
private static class AutoConfigurationGroup
    implements DeferredImportSelector.Group, BeanClassLoaderAware, BeanFactoryAware, ResourceLoaderAware {
  @Override
  public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
    // 确定是 AutoConfigurationImportSelector 类型
    Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
        () -> String.format("Only %s implementations are supported, got %s",
            AutoConfigurationImportSelector.class.getSimpleName(),
            deferredImportSelector.getClass().getName()));
    // 调用 getAutoConfigurationEntry 方法获取要引入的配置实体（包含所有处理过的自动配置类）
    AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
        .getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
    // 放入自动配置实体集合中
    this.autoConfigurationEntries.add(autoConfigurationEntry);
    for (String importClassName : autoConfigurationEntry.getConfigurations()) {
      this.entries.putIfAbsent(importClassName, annotationMetadata);
    }
  }
  @Override
  public Iterable<Entry> selectImports() {
    if (this.autoConfigurationEntries.isEmpty()) {
      return Collections.emptyList();
    }
    // 要排除的配置
    Set<String> allExclusions = this.autoConfigurationEntries.stream()
        .map(AutoConfigurationEntry::getExclusions).flatMap(Collection::stream).collect(Collectors.toSet());
    // 处理过的自动配置类
    Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
        .map(AutoConfigurationEntry::getConfigurations).flatMap(Collection::stream)
        .collect(Collectors.toCollection(LinkedHashSet::new));
    // 排除
    processedConfigurations.removeAll(allExclusions);
    // 返回排序过的配置
    return sortAutoConfigurations(processedConfigurations, getAutoConfigurationMetadata()).stream()
        .map((importClassName) -> new Entry(this.entries.get(importClassName), importClassName))
        .collect(Collectors.toList());
  }
}
```

在 process 方法中会调用 getAutoConfigurationEntry 方法获取要引入的配置实体（包含所有处理过的自动配置类）。getAutoConfigurationMetadata 方法的作用是从 `META-INF\spring-autoconfigure-metadata.properties` 文件中获取自动配置的元数据。

```properties
# 只列出了部分内容
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration.ConditionalOnClass=com.google.gson.Gson
org.springframework.boot.autoconfigure.web.reactive.WebFluxAutoConfiguration.AutoConfigureOrder=-2147483638
org.springframework.boot.autoconfigure.freemarker.FreeMarkerServletWebConfiguration.ConditionalOnClass=javax.servlet.Servlet,org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.ConditionalOnClass=javax.servlet.Servlet,org.springframework.web.servlet.config.annotation.WebMvcConfigurer,org.springframework.web.servlet.DispatcherServlet
```

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(
        AutoConfigurationMetadata autoConfigurationMetadata,
        AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    // 获取 EnableAutoConfiguration 注解的属性（exclude、excludeName）
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 获取所有自动配置类的全限定名集合
    List<String> configurations = getCandidateConfigurations(annotationMetadata,
            attributes);
    // 去重
    configurations = removeDuplicates(configurations);
    // 应用 exclusion 属性排除
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    // 根据自动配置元数据来过滤
    configurations = filter(configurations, autoConfigurationMetadata);
    // 现在已经找到所有需要被应用的候选配置类，广播事件 AutoConfigurationImportEvent
    fireAutoConfigurationImportEvents(configurations, exclusions);
    // 构建 AutoConfigurationEntry
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

该方法总的来说是对所有已知的自动配置类进行处理和筛选。

getCandidateConfigurations 方法使用了内部工具类 SpringFactoriesLoader 查找 classpath 下所有 jar 包中的 `META-INF\spring.factories` 文件，读取文件中 key 为 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 的所有配置类的全限定名，保存到集合中。下面是 spring-boot-autoconfigure.jar 包中 `spring.factories` 文件的内容。   

```properties
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudServiceConnectorsAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
# 以下内容过长，省略之
```

# 启动过程分析
要启动 Spring Boot，一个常见的方式就是在 main 方法中调用 SpringApplication 的 run 方法，其中 primarySource 参数可以理解为带有 BeanDefinition 信息的类。一般我们会将带有 `@SpringBootApplication` 注解的类作为 primarySource，同时为了方便，也直接在该类中创建 main 方法。

```java
/**
*   org.springframework.boot.SpringApplication
*   @param primarySource the primary bean source
*   @param args the application arguments（通常是 main 方法的参数）
*/
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
  return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
  return new SpringApplication(primarySources).run(args);
}

public SpringApplication(Class<?>... primarySources) {
  this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  this.resourceLoader = resourceLoader;
  Assert.notNull(primarySources, "PrimarySources must not be null");
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  // 通过尝试加载不同 Web application 的代表性类来确认当前的 Web 环境，比如 Servlet、Reactive
  this.webApplicationType = WebApplicationType.deduceFromClasspath();
  // 从 META-INF/spring.factories 中获取所有的 ApplicationContextInitializer 并进行实例化，保存到 initializers 属性中
  setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
  // 从 META-INF/spring.factories 中获取所有的 ApplicationListener 并进行实例化，保存到 listeners 属性中
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  // 根据方法调用的堆栈找到调用 main 方法的类
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

这个名为 run 的类方法实际上创建了一个 SpringApplication 实例，然后再调用该实例的 run 方法。

```java
/**
*   org.springframework.boot.SpringApplication
*/
public ConfigurableApplicationContext run(String... args) {
  // 主要用来计算启动用时
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  // 容器上下文
  ConfigurableApplicationContext context = null;
  // 错误汇报机制
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
  // 设置 Headless 模式
  configureHeadlessProperty();
  // 从 META-INF/spring.factories 中获取所有的 SpringApplicationRunListener 并传入 args 进行实例化
  SpringApplicationRunListeners listeners = getRunListeners(args);
  // 执行所有 SpringApplicationRunListener 的 starting 方法
  listeners.starting();
  try {
    // 参数
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    // 准备 Environment
    ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
    // 跳过对自定义 BeanInfo 的搜索
    configureIgnoreBeanInfo(environment);
    // Banner 的处理
    Banner printedBanner = printBanner(environment);
    // 创建容器上下文，在 Servlet 环境下创建的是 AnnotationConfigServletWebServerApplicationContext
    context = createApplicationContext();
    // 从 META-INF/spring.factories 中获取所有的 SpringBootExceptionReporter 并传入 context 进行实例化
    exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
        new Class[] { ConfigurableApplicationContext.class }, context);
    // 容器上下文的准备工作
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    // 刷新容器，最终执行的是 AbstractApplicationContext 的 refresh 方法，同时还会注册 shutdownHook
    refreshContext(context);
    // 空方法，可以由子类实现
    afterRefresh(context, applicationArguments);
    // 启动结束，计算用时
    stopWatch.stop();
    // 日志处理
    if (this.logStartupInfo) {
      new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
    }
    // 执行所有 SpringApplicationRunListener 的 started 方法
    listeners.started(context);
    // 执行所有 ApplicationRunner 和 CommandLineRunner 的 run 方法
    callRunners(context, applicationArguments);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, listeners);
    throw new IllegalStateException(ex);
  }

  try {
    // 执行所有 SpringApplicationRunListener 的 running 方法
    listeners.running(context);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, null);
    throw new IllegalStateException(ex);
  }
  return context;
}
```

该方法中的逻辑比较清晰，但是有些逻辑并不是我们要重点关注的，这里只捡着重点的说。在容器上下文创建之前，Spring Boot 会通过 SpringFactoriesLoader 从 `META-INF/spring.factories` 中加载所有的 SpringApplicationRunListener，目前该接口只有一种官方实现：EventPublishingRunListener，它的作用是在容器启动的各个阶段调用对应的方法来广播对应的容器事件。

Spring Boot 在创建容器上下文时，会根据当前的环境（比如 Servlet、Reactive）来创建对应的容器上下文，在 Servlet 环境下创建的容器上下文为：`AnnotationConfigServletWebServerApplicationContext`。接着需要调用 prepareContext 方法来执行上下文的一些准备工作。

```java
/**
*   org.springframework.boot.SpringApplication
*/
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
    SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
  // 设置 Environment
  context.setEnvironment(environment);
  // 后置处理，注册 beanNameGenerator，设置 resourceLoader 等
  postProcessApplicationContext(context);
  // 执行所有 initializer 的 initialize 方法
  applyInitializers(context);
  // 执行所有 listener 的 contextPrepared 方法
  listeners.contextPrepared(context);
  // 启动日志的处理
  if (this.logStartupInfo) {
    logStartupInfo(context.getParent() == null);
    logStartupProfileInfo(context);
  }
  // 添加几个单例 bean
  ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
  beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
  if (printedBanner != null) {
    beanFactory.registerSingleton("springBootBanner", printedBanner);
  }
  if (beanFactory instanceof DefaultListableBeanFactory) {
    ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
  }
  // 加载 sources（primarySources 和 sources）
  Set<Object> sources = getAllSources();
  Assert.notEmpty(sources, "Sources must not be empty");
  load(context, sources.toArray(new Object[0]));
  // 执行所有 listener 的 contextLoaded 方法
  listeners.contextLoaded(context);
}
```

```java
/**
*   org.springframework.boot.SpringApplication
*/
protected void load(ApplicationContext context, Object[] sources) {
  if (logger.isDebugEnabled()) {
    logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
  }
  // 创建 BeanDefinitionLoader
  BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
  if (this.beanNameGenerator != null) {
    loader.setBeanNameGenerator(this.beanNameGenerator);
  }
  if (this.resourceLoader != null) {
    loader.setResourceLoader(this.resourceLoader);
  }
  if (this.environment != null) {
    loader.setEnvironment(this.environment);
  }
  // 执行 load 方法
  loader.load();
}

BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
  Assert.notNull(registry, "Registry must not be null");
  Assert.notEmpty(sources, "Sources must not be empty");
  this.sources = sources;
  // 初始化了一个 AnnotatedBeanDefinitionReader
  this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
  // 初始化了一个 XmlBeanDefinitionReader
  this.xmlReader = new XmlBeanDefinitionReader(registry);
  if (isGroovyPresent()) {
    this.groovyReader = new GroovyBeanDefinitionReader(registry);
  }
  // 初始化了一个 ClassPathBeanDefinitionScanner
  this.scanner = new ClassPathBeanDefinitionScanner(registry);
  this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
}
```


```java
/**
*   org.springframework.boot.BeanDefinitionLoader
*/
public int load() {
  int count = 0;
  for (Object source : this.sources) {
    count += load(source);
  }
  return count;
}

private int load(Object source) {
  Assert.notNull(source, "Source must not be null");
  // source 是 Class，使用 AnnotatedBeanDefinitionReader 加载 BeanDefinition
  if (source instanceof Class<?>) {
    return load((Class<?>) source);
  }
  // source 是 Resource，使用 XmlBeanDefinitionReader 加载 BeanDefinition
  if (source instanceof Resource) {
    return load((Resource) source);
  }
  // source 是 Package，使用 ClassPathBeanDefinitionScanner 扫描
  if (source instanceof Package) {
    return load((Package) source);
  }
  // source 是字符串，解析字符串，然后尝试使用多种方式来处理
  if (source instanceof CharSequence) {
    return load((CharSequence) source);
  }
  throw new IllegalArgumentException("Invalid source type " + source.getClass());
}

private int load(Class<?> source) {
  if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
    // Any GroovyLoaders added in beans{} DSL can contribute beans here
    GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
    load(loader);
  }
  // 注解中是否包含 Component 注解（Configuration 注解也算）
  if (isComponent(source)) {
    // 将 source 作为 BeanDefinition 注册到容器中
    this.annotatedReader.register(source);
    return 1;
  }
  return 0;
}
```

准备工作结束以后，需要执行 refreshContext 方法。在该方法中，实际执行的是 AbstractApplicationContext 的 refresh 方法，与此同时还会注册 shutdownHook。其余的工作由于不是重点，这里不再详细说明。

# main 方法不退出的秘密
也许你会好奇，在 main 方法执行完毕之后，Spring Boot 为什么没有退出呢？我们知道，使用 `System.exit()` 或 `Runtime.exit()` 可以使当前 JVM 进程退出。而另一个会导致 JVM 进程退出的原因是所有的非 daemon 线程完全终止，如果根据这个条件反推的话，只要保证 Spring Boot 进程中包含至少一个非 daemon 线程一直运行，就可以保证程序不会退出。

通过跟踪代码可以发现，在调用 refresh 方法时，会调用一个叫做 onRefresh 的方法，该方法是一个空方法，由子类实现。由于在 Servlet 环境下创建的容器上下文是：`AnnotationConfigServletWebServerApplicationContext`，实际实现 onRefresh 方法的是它的父类：`ServletWebServerApplicationContext`。在该类中，onRefresh 方法会创建一个 WebServer。

```java
protected void onRefresh() {
  super.onRefresh();
  try {
    createWebServer();
  }
  catch (Throwable ex) {
    throw new ApplicationContextException("Unable to start web server", ex);
  }
}
```

默认情况下，这个 WebServer 也就是 Tomcat，这里最终会调用 TomcatWebServer 的 initialize 方法。

```java
private void initialize() throws WebServerException {
  logger.info("Tomcat initialized with port(s): " + getPortsDescription(false));
  synchronized (this.monitor) {
    try {
      addInstanceIdToEngineName();
      Context context = findContext();
      context.addLifecycleListener((event) -> {
        if (context.equals(event.getSource()) && Lifecycle.START_EVENT.equals(event.getType())) {
          // Remove service connectors so that protocol binding doesn't
          // happen when the service is started.
          removeServiceConnectors();
        }
      });
      // Start the server to trigger initialization listeners
      this.tomcat.start();
      // We can re-throw failure exception directly in the main thread
      rethrowDeferredStartupExceptions();
      try {
        ContextBindings.bindClassLoader(context, context.getNamingToken(), getClass().getClassLoader());
      }
      catch (NamingException ex) {
        // Naming is not enabled. Continue
      }
      // Unlike Jetty, all Tomcat threads are daemon threads. We create a
      // blocking non-daemon to stop immediate shutdown
      startDaemonAwaitThread();
    }
    catch (Exception ex) {
      stopSilently();
      destroySilently();
      throw new WebServerException("Unable to start embedded Tomcat", ex);
    }
  }
}

private void startDaemonAwaitThread() {
  Thread awaitThread = new Thread("container-" + (containerCounter.get())) {
    @Override
    public void run() {
      TomcatWebServer.this.tomcat.getServer().await();
    }
  };
  awaitThread.setContextClassLoader(getClass().getClassLoader());
  awaitThread.setDaemon(false);
  awaitThread.start();
}
```

在 initialize 方法中，调用了一个名为 `startDaemonAwaitThread` 的方法，从方法的注释可以看出：与 Jetty 这些容器不同，Tomcat 的所有线程都是守护线程。所以该方法创建了一个非 daemon 的线程，这个线程会一直检测 stopAwait 变量，在这个变量值为 false 时会直接休眠（调用 `Thread.sleep(10000)` 方法）。只有在 Tomcat 收到终止运行的信号时，该变量的值才会改变，在该线程从休眠中苏醒时会检测变量并结束运行。由于 JVM 进程中所有的非 daemon 线程都结束了运行，此时 JVM 进程也会退出。