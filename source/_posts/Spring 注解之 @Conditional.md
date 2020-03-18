---
title: Spring 注解之 @Conditional
date: 2020/3/18 12:27:0
tags: [Spring]
categories: [Spring]
---

`@Conditional` 是 Spring 4 提供的新注解，它的作用是按照一定的条件进行判断，当满足条件时会给容器注册 bean。举个例子，比如说我们有一个接口，这个接口有多个实现类，当我们将这个接口交给 Spring 容器管理时通常只能选择其中一个作为实现类，但是我们又希望能够根据不同的情况注册不同的实现类，此时就可以使用该注解。

<!--more-->

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

    /**
     * All {@link Condition Conditions} that must {@linkplain Condition#matches match}
     * in order for the component to be registered.
     */
    Class<? extends Condition>[] value();

}
```

可以看到，该注解可以在类、接口、注解、枚举以及方法上使用，并且该注解在编译和运行时都有效。该注解在使用时需要传入一个 Class 数组，数组元素的类型要求必须是 `Condition` 类型或它的子类。接下来我们查看 `Condition` 接口：

```java
@FunctionalInterface
public interface Condition {

    /**
     * Determine if the condition matches.
     * @param context the condition context
     * @param metadata metadata of the {@link org.springframework.core.type.AnnotationMetadata class}
     * or {@link org.springframework.core.type.MethodMetadata method} being checked
     * @return {@code true} if the condition matches and the component can be registered,
     * or {@code false} to veto the annotated component's registration
     */
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```

可以看到，这是一个函数式接口，该接口有一个 matches 方法，其中 context 是 Condition 的上下文，包含 BeanDefinitionRegistry、BeanFactory、Environment 等信息，AnnotatedTypeMetadata 是使用了 `@Conditional` 注解的类型或方法的元数据，比如有一个类 WebMvcAutoConfiguration，它使用了 `@Conditional` 注解，那么 metadata 参数就是 WebMvcAutoConfiguration 的元数据。

在 Spring 的整个体系当中，广泛使用 `@Conditional` 注解家族成员的就是 Spring Boot，因为 Spring Boot 包含大量的自动配置类，它们需要根据不同的条件选择性地注册到 Spring 容器当中。因此接下来我们就根据 Spring Boot 继续向下分析，先查看实现了 Condition 接口的类 SpringBootCondition。

```java
public abstract class SpringBootCondition implements Condition {

    private final Log logger = LogFactory.getLog(getClass());

    @Override
    public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 根据元数据获取当前类或者方法的名称
    	String classOrMethodName = getClassOrMethodName(metadata);
    	try {
            // 获取条件匹配的结果
            ConditionOutcome outcome = getMatchOutcome(context, metadata);
            // 打印日志
            logOutcome(classOrMethodName, outcome);
            // 暂存结果数据
            recordEvaluation(context, classOrMethodName, outcome);
            // 返回结果
            return outcome.isMatch();
    	}
    	catch (NoClassDefFoundError ex) {
            throw new IllegalStateException("Could not evaluate condition on " + classOrMethodName + " due to "
                    + ex.getMessage() + " not " + "found. Make sure your own configuration does not rely on "
                    + "that class. This can also happen if you are "
                    + "@ComponentScanning a springframework package (e.g. if you "
                    + "put a @ComponentScan in the default package by mistake)", ex);
    	}
    	catch (RuntimeException ex) {
            throw new IllegalStateException("Error processing condition on " + getName(metadata), ex);
    	}
    }

    private void recordEvaluation(ConditionContext context, String classOrMethodName, ConditionOutcome outcome) {
    	if (context.getBeanFactory() != null) {
            ConditionEvaluationReport.get(context.getBeanFactory()).recordConditionEvaluation(classOrMethodName, this,
                    outcome);
    	}
    }

    public abstract ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

SpringBootCondition 的实现类有很多，我们这里挑选一个比较常见的实现类 OnClassCondition。

```java
class OnClassCondition extends FilteringSpringBootCondition {

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
        ClassLoader classLoader = context.getClassLoader();
        ConditionMessage matchMessage = ConditionMessage.empty();
        // 通过 metadata 获取 ConditionalOnClass 注解中的值
        List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
        if (onClasses != null) {
            // 通过使用类加载器尝试加载注解中的值来判断该值是否存在，如果没有则暂存
            List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
            if (!missing.isEmpty()) {
                // 创建不匹配的结果
                return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
                        .didNotFind("required class", "required classes").items(Style.QUOTE, missing));
            }
            // 创建匹配的结果
            matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
                    .found("required class", "required classes")
                    .items(Style.QUOTE, filter(onClasses, ClassNameFilter.PRESENT, classLoader));
        }
        List<String> onMissingClasses = getCandidates(metadata, ConditionalOnMissingClass.class);
        if (onMissingClasses != null) {
            List<String> present = filter(onMissingClasses, ClassNameFilter.PRESENT, classLoader);
            if (!present.isEmpty()) {
                return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnMissingClass.class)
                        .found("unwanted class", "unwanted classes").items(Style.QUOTE, present));
            }
            matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
                    .didNotFind("unwanted class", "unwanted classes")
                    .items(Style.QUOTE, filter(onMissingClasses, ClassNameFilter.MISSING, classLoader));
        }
        return ConditionOutcome.match(matchMessage);
    }
}
```

通过以上代码我们可以看出，`@ConditionalOnClass` 在注解值中所有的类都存在时（通过尝试使用类加载器加载指定的类的方式判断）才会匹配，此时也就意味着使用该注解的类将被注册到容器中。同理还有很多类似的注解，比如 `ConditionalOnMissingClass` 会在注解中所有的值都不存在时才会匹配，`@ConditionalOnBean` 会在注解中所有的值都在容器中存在时才会匹配，`@ConditionalOnProperty` 注解稍微复杂一点，它包含 prefix、name、havingValue、matchIfMissing 等属性，其中 prefix 表示配置文件中配置的前缀，name 表示具体的配置属性名称，havingValue 表示属性的值，matchIfMissing 表示如果所有的值都不满足时是否匹配。举个例子，假如我们的配置为：

```yml
# storage
storage:
  # 默认使用本机文件系统
  type: filesystem
  filesystem:
    # 图片存放目录
    imgFolder: ${user.home}/saber/
    # 图片缓存时间控制，可以带单位
    cacheTime: 7d
```

我们的注解为 `@ConditionalOnProperty(name = "storage.type", havingValue = "filesystem")`，那么使用这个注解的类就可以被加载到容器当中。

大概说完了 `@Conditional` 注解，到了这里我们还有一个疑惑就是，Spring 是在什么时候什么地方处理 `@Conditional` 注解的呢？答案就是在 ConfigurationClassParser 类中，具体的位置是在 doProcessConfigurationClass 方法中，这里就不具体展开了。