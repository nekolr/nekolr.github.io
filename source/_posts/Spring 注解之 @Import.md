---
title: Spring 注解之 @Import
date: 2017/7/22 10:47:0
tags: [Spring]
categories: [Spring]
---

在 Spring 中，使用 `<import>` 标签或者 `@Import` 注解能够向容器中导入配置类（使用 `@Configuration` 注解的类），利用这个功能可以很方便地实现模块化配置的效果。从 Spring 官方文档给出的说明了解到，在 Spring 4.2 版本以前，`@Import` 注解只支持导入配置类，在 Spring 4.2 及以后的版本，该注解支持导入普通的 Java 类。

<!--more-->

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
    Class<?>[] value();
}
```

可以看出，`@Import` 注解可以放在类、接口、注解和枚举上，我们简单写点代码来试验一下。

```java
@Configuration
public class ConfigA {
    @Bean
    public A getA() {
        return new A();
    }
}
```

```java
public class ConfigB {
    @Bean
    public B getB() {
        return new B();
    }
}
```

```java
@Import({ConfigA.class, ConfigB.class})
public class ConfigC {
}
```

```java
public class Main {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(ConfigC.class);
        A a = context.getBean(A.class);
        B b = context.getBean(B.class);
    }
}
```

除了导入配置类，`@Import` 注解还有一些更高级的用法，那就是导入实现了 ImportBeanDefinitionRegistrar 接口或者 ImportSelector 接口的类。在这之前，我们需要知道 Spring 容器最终是通过 ConfigurationClassParser 工具类来解析该注解的。

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
        // 导入的类实现了 ImportSelector 接口
        if (candidate.isAssignable(ImportSelector.class)) {
          Class<?> candidateClass = candidate.loadClass();
          ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
          ParserStrategyUtils.invokeAwareMethods(
              selector, this.environment, this.resourceLoader, this.registry);
          if (selector instanceof DeferredImportSelector) {
            this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
          }
          else {
            // 调用接口的 selectImports 方法
            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
            processImports(configClass, currentSourceClass, importSourceClasses, false);
          }
        }
        // 导入的类实现了 ImportBeanDefinitionRegistrar 接口
        else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
          Class<?> candidateClass = candidate.loadClass();
          ImportBeanDefinitionRegistrar registrar =
              BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
          ParserStrategyUtils.invokeAwareMethods(
              registrar, this.environment, this.resourceLoader, this.registry);
          configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
        }
        else {
          // 当做配置类处理
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

如果导入的是一个 ImportSelector，那么在这里会调用它的 selectImports 方法，加载多个类到 Spring 容器中，典型的比如 Spring Boot 的 `@EnableAutoConfiguration` 自动装配注解。

如果导入的是一个 ImportBeanDefinitionRegistrar，那么会在这里将它添加到配置类的 importBeanDefinitionRegistrars 属性中。Spring 容器在初始化时，会通过 ConfigurationClassBeanDefinitionReader 的 loadBeanDefinitions 方法获取所有的 BeanDefinition：

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
  TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
  for (ConfigurationClass configClass : configurationModel) {
    loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
  }
}

private void loadBeanDefinitionsForConfigurationClass(
    ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

  if (trackedConditionEvaluator.shouldSkip(configClass)) {
    String beanName = configClass.getBeanName();
    if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
      this.registry.removeBeanDefinition(beanName);
    }
    this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
    return;
  }

  if (configClass.isImported()) {
    registerBeanDefinitionForImportedConfigurationClass(configClass);
  }
  for (BeanMethod beanMethod : configClass.getBeanMethods()) {
    loadBeanDefinitionsForBeanMethod(beanMethod);
  }

  loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
  // 调用 ImportBeanDefinitionRegistrar 的 registerBeanDefinitions 方法
  loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

很多第三方框架在集成 Spring 的时候，都会通过实现 ImportBeanDefinitionRegistrar 接口将自定义的 BeanDefinition 注册到 Spring 容器中，比如 MyBatis 的 `@MapperScan` 注解。