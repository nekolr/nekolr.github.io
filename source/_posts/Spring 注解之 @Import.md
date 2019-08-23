---
title: Spring 注解之 @Import
date: 2017/7/22 10:47:0
tags: [Spring]
categories: [Spring]
---
在 Spring 的配置文件中可以使用 `<import/>` 标签来模块化配置文件，很自热的就想到使用注解也能够实现模块化的配置。		
<!--more-->		
		
@Import 源码

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
    Class<?>[] value();
}
```
可以看出，`@Import` 注解可以放在类、接口、注解和枚举上。从 Spring 官方文档给出的说明了解到，在 Spring 4.2 版本以前，`@Import` 注解只支持导入配置类（使用 `@Configuration` 注解的类）；在 Spring 4.2 及以后的版本 `@Import` 支持导入普通的 Java 类。		

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
在 Spring 中，使用 `@Configuration` 注解的类其实也是 Spring 容器中的一个 Bean，因此同样可以使用 `@Autowired` 和 `@Value` 等注解