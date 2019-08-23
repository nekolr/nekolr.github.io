---
title: Spring 工具类之 ConfigurationClassParser
date: 2019/5/5 10:41:0
tags: [Spring]
categories: [Spring]
---
ConfigurationClassParser 类主要用于分析带有 @Configuration 注解的配置类，产生 ConfigurationClass 配置类对象，并将这些配置类对象传递给调用者（使用该工具的类为 ConfigurationClassPostProcessor，它是一个 BeanFactoryPostProcessor，会在 Spring 容器启动时被调用）。  

<!--more-->

对于其他使用非 @Configuration 注解的 bean，比如使用了 @Component 注解的类，它会使用另外一个工具类 ComponentScanAnnotationParser 来分析扫描，并且会直接将扫描到的 bean 定义注册到 Spring 容器当中。  

ConfigurationClassParser 的外部调用入口为 parse 方法。  

```java
/**
* configCandidates 是一组需要被分析的配置类的集合
*/
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

    for (BeanDefinitionHolder holder : configCandidates) {
        // 获取 BeanDefinition
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                // 是 AnnotatedBeanDefinition
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                // 是 AbstractBeanDefinition 并且指定了 beanClass
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

    processDeferredImportSelectors();
}

protected final void parse(String className, String beanName) throws IOException {
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

processConfigurationClass 方法会分析一个配置类。该方法从配置类开始遍历其所有需要处理的父类，每个类都调用 doProcessConfigurationClass 来处理，直到该类已经被处理过或者该类为 JDK 提供的类（即类的全限定名以 java 开头，比如 java.lang.Object）。所有被处理过的类都会保存到 this.configurationClasses 中。  

```java
/**
* 分析配置类
*/
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }
    // 根据当前提供的配置类去已经处理过的配置类集合中寻找
    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    if (existingClass != null) {
        // 如果配置类是某些类通过 @Import 导入的
        if (configClass.isImported()) {
            // 如果已经处理过的该配置类同样也是其他类通过 @Import 导入的
            if (existingClass.isImported()) {
                // 合并两者的 importedBy
                existingClass.mergeImportedBy(configClass);
            }
            // Otherwise ignore new imported config class; existing non-imported class overrides it.
            return;
        }
        else {
            // Explicit bean definition found, probably replacing an import.
            // Let's remove the old one and go with the new one.
            this.configurationClasses.remove(configClass);
            for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext();) {
                if (configClass.equals(it.next())) {
                    it.remove();
                }
            }
        }
    }

    // Recursively process the configuration class and its superclass hierarchy.
    // 从当前配置类 configClass 开始向上沿着类继承结构逐层执行 doProcessConfigurationClass，
    // 直到遇到父类是由 JDK 提供的类时结束循环
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}
```

doProcessConfigurationClass 才是真正处理配置类的方法，具体的处理如下：  

- 配置类的成员类也可能是配置类，所以需要先遍历这些成员配置类，然后分别调用 doProcessConfigurationClass 方法。
- 处理配置类上的 @PropertySources 和 @PropertySource 注解。
- 处理配置类上的 @ComponentScans 和 @ComponentScan 注解。
- 处理配置类上的 @Import 注解。
- 处理配置类上的 @ImportResource 注解。
- 处理配置类中每个带有 @Bean 注解的方法。
- 处理配置类所实现的接口的缺省方法。
- 检查父类是否需要处理，如果父类需要处理则该方法会返回父类，否则返回 null。

> 返回父类表示当前配置类尚未处理完成，processConfigurationClass 方法会继续处理其父类；返回 null 才表示该配置类的已经处理完成。  

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {

    // Recursively process any member (nested) classes first
    processMemberClasses(configClass, sourceClass);

    // Process any @PropertySource annotations
    // 处理 @PropertySources 注解，解析属性文件，将解析出来的属性资源添加到 environment
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    // 处理 @ComponentScan 注解
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    // 处理 @Import 注解
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // Process any @ImportResource annotations
    // 处理 @ImportResource 注解。获取 @ImportResource 注解的 locations 属性，得到资源文件的地址，
    // 然后遍历这些资源文件并把它们添加到配置类的 importedResources 属性中
    if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
        AnnotationAttributes importResource =
                AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    // 处理 @Bean注解，获取被 @Bean 注解修饰的方法，然后添加到配置类的 beanMethods 属性中
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}
```

```java
/**
* 处理配置类上搜集到的 @Import 注解
*
* 参数 configuClass 配置类
* 参数 currentSourceClass 当前源码类
* 参数 importCandidates 所有的 @Import 注解的 value
* 参数 checkForCircularImports 是否检查循环导入
*/
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        // 如果检测到了循环依赖，则报告错误
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            // 循环处理所有的 @Import
            for (SourceClass candidate : importCandidates) {
                // 如果使用 @Import 导入的是 ImportSelector 类型的类
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            selector, this.environment, this.resourceLoader, this.registry);
                    if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectors.add(
                                new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                    }
                    else {
                        // 调用 selectImports 方法
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                        processImports(configClass, currentSourceClass, importSourceClasses, false);
                    }
                }
                // 如果使用 @Import 导入的是 ImportBeanDefinitionRegistrar 类型的类
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            registrar, this.environment, this.resourceLoader, this.registry);
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    // 其他类型都当作配置类处理，也就是相当于使用了 @Configuration 注解的配置类
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```

# 参考
> [Spring 工具类 ConfigurationClassParser 分析得到配置类](https://blog.csdn.net/andy_zhang2007/article/details/78549773 )
