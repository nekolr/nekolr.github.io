---
title: Spring IoC 容器剖析
date: 2018/5/27 19:08:0
tags: [Spring]
categories: [Spring]
---

> 使用 spring 4.3.16.RELEASE 分析

整个 IoC 容器都是围绕着 `BeanFactory` 和 `ApplicationContext` 来设计的。`BeanFactory` 提供容器的基本功能；`ApplicationContext` 继承自 `BeanFactory`，不光实现了容器的基本功能，还实现了一些更高级的容器特性。  

<!--more-->		

# BeanFactory 容器设计

![ConfigurableBeanFactory](https://img.nekolr.com/images/2018/05/27/3q7.png)

这是一条主要的`BeanFactory`设计路线。  

- `BeanFactory` 提供了如 `getBean` 这样的基本方法用来获取容器中的 bean。
- `HierarchicalBeanFactory` 提供了 `getParentBeanFactory` 方法，使得容器获得了父子容器管理的功能。
- `SingletonBeanRegistry` 提供了管理单例 bean 的功能。
- `ConfigurableBeanFactory` 提供了一些容器的配置功能，比如设置容器的父容器的 `setParentBeanFactory` 方法。

在这条路线上，有一个实现了基本容器功能的类 `DefaultListableBeanFactory`。  

![DefaultListableBeanFactory](https://img.nekolr.com/images/2018/05/27/o6x.png)

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

![ApplicationContext](https://img.nekolr.com/images/2018/05/27/WnD.png)

IoC 容器的第二条设计路线以 `ApplicationContext` 接口为主，在继承了 `BeanFactory` 的同时，还继承了 `MessageSource`、`ApplicationEventPublisher`、`ResourceLoader` 等接口。  

![ConfigurableWebApplicationContext](https://img.nekolr.com/images/2018/05/27/YAN.png)

 `ApplicationContext` 作为 `BeanFactory` 的实现，和 `XmlBeanFactory` 一样，也是在 `DefaultListableBeanFactory` 这个基本的容器实现上做扩展。我们常用的应用上下文基本上都是 `ConfigurableApplicationContext` 或 `WebApplicationContext` 的实现。  

实现了 `WebApplicationContext` 接口的，比如：  

- 可以通过加载纯注解的配置类而初始化容器的 `AnnotationConfigWebApplicationContext`。  
- Web 应用中，我们一般会在 web.xml 中定义监听器 `org.springframework.web.context.ContextLoaderListener`，它实际上使用的是 `XmlWebApplicationContext`。  

实现了 `ConfigurableApplicationContext` 接口的，比如：  

- `FileSystemXmlApplicationContext`，通过文件系统加载 xml 配置文件来初始化容器。  
- `ClassPathXmlApplicationContext`，通过 classpath 加载 xml 配置文件来初始化容器。  
- `AnnotationConfigApplicationContext`，通过加载纯注解的配置类来初始化容器。  

# 容器初始化过程

IoC 容器的初始化过程主要是通过 `AbstractApplicationContext` 的 `refresh` 方法完成。  

![AbstractApplicationContext](https://img.nekolr.com/images/2018/05/28/LXL.png)

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
            *   执行在上下文注册为 bean 的 BeanFactoryPostProcessor 们。
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
            *   初始化事件广播者。
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

Spring 在刷新 BeanFactory 时，检测 BeanFactory 是否已经创建，已经创建会执行销毁方法，然后重新创建。在创建 BeanFactory 时，解析 bean 的配置。  

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
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
        destroyBeans();
        closeBeanFactory();
    }
    try {
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

在 bean 的实例化和初始化过程中会执行一些回调方法，具体的过程如下：  

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Initialize conversion service for this context.
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
            @Override
            public String resolveStringValue(String strVal) {
                return getEnvironment().resolvePlaceholders(strVal);
            }
        });
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();

    // 这里实例化和初始化非懒加载的单例 bean
    beanFactory.preInstantiateSingletons();
}
```

```java
/**
*   org.springframework.beans.factory.support.DefaultListableBeanFactory
*/
@Override
public void preInstantiateSingletons() throws BeansException {
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Pre-instantiating singletons in " + this);
    }

    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                        @Override
                        public Boolean run() {
                            return ((SmartFactoryBean<?>) factory).isEagerInit();
                        }
                    }, getAccessControlContext());
                }
                else {
                    isEagerInit = (factory instanceof SmartFactoryBean &&
                            ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                if (isEagerInit) {
                    // 获取 bean
                    getBean(beanName);
                }
            }
            else {
                // 获取 bean
                getBean(beanName);
            }
        }
    }
    // Trigger post-initialization callback for all applicable beans...
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

```java
/**
*   org.springframework.beans.factory.support.AbstractBeanFactory
*/
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(
        final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
        throws BeansException {

    final String beanName = transformedBeanName(name);
    Object bean;

    // Spring 手动注册了一些单例 bean，这里检测是不是这些 bean。
    // 如果是，那么再检测是不是工厂 bean，如果是返回其工厂方法返回的实例，如果不是返回 bean 本身。
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isDebugEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 如果存在父容器，并且父容器中存在此 bean 定义，那么交由其父容器初始化
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }

        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // 依赖关系处理
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 检测是否存在循环依赖
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    // 将依赖关系保存，方便容器销毁时，销毁 bean 之前需要先销毁依赖的 bean
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // 创建 bean 实例（singleton）
            if (mbd.isSingleton()) {
                // 检测是否已经存在
                sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        try {
                            // 创建 bean 实例的方法
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
            // 创建 bean 实例（prototype）
            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                        @Override
                        public Object getObject() throws BeansException {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
        try {
            return getTypeConverter().convertIfNecessary(bean, requiredType);
        }
        catch (TypeMismatchException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

```java
/**
*   org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*/
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
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
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        // 此处进行 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation() 回调
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }
    // 执行 bean 的实例化操作
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isDebugEnabled()) {
        logger.debug("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
}
```

```java
/**
*   org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*/
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 创建 bean 实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;

    // Allow post-processors to modify the merged bean definition.
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

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    // Initialize the bean instance.
    // 初始化 bean（实例化之后，属性注入之前）
    Object exposedObject = bean;
    try {
        /**
        *   填充 bean
        *   
        *   1 调用 InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
        *   2 自动注入属性
        */
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            // 初始化 bean
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
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
                            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    // 注册 DisposableBean 接口
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

```java
/**
*   org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*/
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // Make sure bean class is actually resolved at this point.
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    if (mbd.getFactoryMethodName() != null)  {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // Shortcut when re-creating the same bean...
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            // 实例化 bean
            return instantiateBean(beanName, mbd);
        }
    }

    // Need to determine the constructor...
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // No special handling: simply use no-arg constructor.
    // 简单使用无参构造器进行实例化
    return instantiateBean(beanName, mbd);
}

protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        if (System.getSecurityManager() != null) {
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    return getInstantiationStrategy().instantiate(mbd, beanName, parent);
                }
            }, getAccessControlContext());
        }
        else {
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
    }
}
```

```java
/**
*   org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
*   初始化 bean
*/
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                invokeAwareMethods(beanName, bean);
                return null;
            }
        }, getAccessControlContext());
    }
    else {
        // 调用 Aware 一类的接口方法
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 调用 InstantiationAwareBeanPostProcessor.postProcessBeforeInitialization()
        // 或 BeanPostProcessor.postProcessBeforeInitialization()
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 调用 InitializingBean.afterPropertiesSet() 或 init-method 属性指定的方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 调用 InstantiationAwareBeanPostProcessor.postProcessAfterInitialization()
        // 或 BeanPostProcessor.postProcessAfterInitialization()
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}

private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

# Spring Bean 生命周期

在 Spring bean 的完整生命周期中，Spring 提供了一系列的扩展方法方便我们调用，可以将这些方法大体划分为：  

- bean 级别的方法
这个包括 bean 本身的方法，通过配置文件定义的 `init-method` 方法和 `destroy-method` 方法，bean 实现 `InitializingBean`、`BeanNameAware`、`BeanFactoryAware`、`DisposableBean` 等接口的方法。  

- 容器级别的方法
这个包括 bean 后置处理器 `BeanPostProcessor` 和 `InstantiationAwareBeanPostProcessor` 接口的方法，bean 工厂后置处理器 `BeanFactoryPostProcessor` 接口的方法。  

![Spring bean 生命周期 ](https://img.nekolr.com/images/2018/05/31/dQz.png)  

# 参考

> 《Spring 技术内幕：深入解析 Spring 架构与计原理 (第 2 版)》

