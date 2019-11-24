---
title: Spring 事务
date: 2019/10/15 10:55:0
tags: [Spring]
categories: [Spring]
---
传统上，Java 开发人员使用事务有两种选择：本地事务和全局事务，这两种选择都有很大的局限性。

<!--more-->

在全局事务中我们可以使用多个事务资源，比如数据库和消息队列。Web 服务器通过 JTA 管理全局事务，但是 JTA 的 API 异常繁琐，且它的 UserTransaction 需要通过 JNDI 产生，这意味着我们需要使用 JNDI 才能使用 JTA。在以前，使用全局事务的首选方法是 EJB CMT（容器管理的事务），CMT 使用声明式的事务管理，尽管使用 EJB 本身需要使用 JNDI，但是 EJB CMT 消除了与事务相关的 JNDI 的查找，并且消除了大多数需要手工编写来控制事务的代码。遗憾的是，CMT 与 JTA 和 Web 容器的环境捆绑在了一起，使用 CMT 就必须使用 JTA 和指定的容器环境。

而本地事务是特定于资源的，比如与 JDBC 连接绑定的事务。本地事务可能更易于使用，但是缺点也很明显：不能跨多个事务资源工作，比如使用 JDBC 连接管理事务的代码不能在全局 JTA 事务中运行；另一个缺点就是侵入了代码模型。

Spring 解决了全局事务和本地事务的弊端，它可以在不同的环境中使用一致的编程模型，只需要编写一次代码，就可以在不同的环境中使用不同的事务管理策略。

# 核心接口
![核心接口](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242036/2019/10/15/pla.png)

## 事务管理器
Spring 并不直接管理事务，而是提供了多种事务管理器，事务管理器将事务委托给具体的持久化相关平台框架（比如 Hibernate、JTA、JDBC 等）来完成。事务管理器的核心接口为 `PlatformTransactionManager`，所有的事务管理器都需要实现该接口。

```java
public interface PlatformTransactionManager {

    /**
    * 通过 TransactionDefinition 得到 TransactionStatus
    */
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException;

    /**
    * 事务提交
    */
    void commit(TransactionStatus status) throws TransactionException;

    /**
    * 事务回滚
    */
    void rollback(TransactionStatus status) throws TransactionException;

}
```

如果我们直接使用 JDBC 来进行持久化，那么需要使用 `DataSourceTransactionManager` 来处理事务，可能的配置为：

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
```

同理，如果使用 Hibernate，则需要使用 `HibernateTransactionManager` 来处理事务；如果使用 Java 持久化标准，也就是 JPA 的话，那么需要使用 `JpaTransactionManager` 来处理事务等等。

# 事务定义
TransactionDefinition 定义了一些事务的基本属性，具体来说包括：事务的传播行为、事务的隔离级别、事务的超时时间以及事务是否只读等。

```java
public interface TransactionDefinition {
    /**
    * 返回事务的传播行为
    */
    int getPropagationBehavior();
    /**
    * 返回事务的隔离级别
    */
    int getIsolationLevel();
    /**
    * 返回事务的超时时间，事务必须在超时时间内完成
    */
    int getTimeout();
    /**
    * 返回事务是否只读
    */
    boolean isReadOnly();
}
```

## 传播行为
当一个事务方法被另一个事务方法调用时，必须指定事务应该如何传播。比如：可能会在现有的事务中运行，也有可能重新开启一个新的事务。Spring 定义了七种事务的传播行为：

| 传播行为 | 描述 |
| ------------ | ------------ |
| PROPAGATION_REQUIRED | 当前方法必须运行在事务中。如果当前事务存在，方法将会在该事务中运行。否则，会启动一个新的事务 |
| PROPAGATION_SUPPORTS | 当前方法不需要事务上下文，但是如果存在当前事务的话，那么该方法会在这个事务中运行 |
| PROPAGATION_MANDATORY | 当前方法必须在事务中运行，如果当前事务不存在，则会抛出一个异常 |
| PROPAGATION_REQUIRED_NEW | 当前方法必须运行在它自己的事务中，即总是会开启一个新的事务。如果存在当前事务，在该方法执行期间，当前事务会被挂起。如果使用 JTATransactionManager 的话，需要访问TransactionManager |
| PROPAGATION_NOT_SUPPORTED | 当前方法不应该运行在事务中。如果存在当前事务，在该方法运行期间，当前事务将被挂起。如果使用 JTATransactionManager 的话，则需要访问 TransactionManager |
| PROPAGATION_NEVER | 当前方法不应该运行在事务中。如果当前有一个事务在运行，则会抛出异常 |
| PROPAGATION_NESTED | 如果当前已经存在一个事务，那么该方法将会在嵌套事务中运行。嵌套的事务可以独立于当前事务进行单独地提交或回滚。如果当前事务不存在，那么其行为与 PROPAGATION_REQUIRED 一样。**各厂商对这种传播行为的支持是有所差异的。** |

下面针对各个传播行为进行具体分析。

### PROPAGATION_REQUIRED
```java
@Transactional(propagation = Propagation.REQUIRED)
methodA() {
    methodB();
}

@Transactional(propagation = Propagation.REQUIRED)
methodB() {

}
```

使用 Spring 的声明式事务注解，Spring 会通过 AOP 的方式使用获取连接、连接提交和连接回滚的操作包裹具体的业务逻辑。当调用 methodA 时，会创建一个事务，遇到 methodB 的调用时，因为已经存在一个事务上下文，所以就将 methodB 加入到当前事务中。

### PROPAGATION_SUPPORTS
```java
@Transactional(propagation = Propagation.REQUIRED)
methodA() {
    methodB();
}

@Transactional(propagation = Propagation.SUPPORTS)
methodB() {

}
```

单独调用 methodB 时，methodB 是在非事务环境下执行的。当调用 methodA 时，methodB 则加入到 methodA 的事务当中。

### PROPAGATION_MANDATORY
```java
@Transactional(propagation = Propagation.REQUIRED)
methodA() {
    methodB();
}

@Transactional(propagation = Propagation.MANDATORY)
methodB() {

}
```

单独调用 methodB 会抛出一个异常。当调用 methodA 时，methodB 会加入到 methodA 的事务当中执行。

### PROPAGATION_REQUIRED_NEW
```java
@Transactional(propagation = Propagation.REQUIRED)
methodA() {
    doSomethingA();
    methodB();
    doSomethingB();
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
methodB() {

}
```

调用方法 methodA 时，相当于调用以下代码：

```java
TransactionManager transactionManager = null;
try {
    // 获得一个 JTA 事务管理器
    transactionManager = getTransactionManager();
    // 开启一个新的事务
    transactionManager.begin();
    Transaction ts1 = transactionManager.getTransaction();
    doSomethingA();
    // 挂起当前事务
    transactionManager.suspend();
    try {
        // 开启第二个事务
        transactionManager.begin();
        Transaction ts2 = transactionManager.getTransaction();
        methodB();
        // 提交第二个事务
        ts2.commit();
    } catch (RunTimeException ex) {
        ts2.rollback();
    } finally {
        // 释放资源
    }
    // methodB 执行完后，恢复第一个事务
    transactionManager.resume(ts1);
    doSomethingB();
    // 提交第一个事务
    ts1.commit();
} catch (RunTimeException ex) {
    ts1.rollback();
} finally {
    // 释放资源
}
```

ts2 是否成功并不依赖于 ts1。如果 methodA 方法在调用 methodB 方法后的 doSomethingB 方法失败了，methodB 方法所做的结果依然被提交，除了 methodB 之外的其它代码导致的结果会被回滚。PROPAGATION_REQUIRES_NEW 这一传播等级尤其适用于 JtaTransactionManager 事务管理器。

### PROPAGATION_NOT_SUPPORTED
总是非事务地执行，并挂起任何存在的事务。PROPAGATION_NOT_SUPPORTED 这一传播等级尤其适用于 JtaTransactionManager 事务管理器。

### PROPAGATION_NEVER
总是非事务地执行，如果存在一个活动的事务，则抛出异常。

### PROPAGATION_NESTED
如果一个活动的事务存在，则运行在一个嵌套的事务中。如果没有活动事务, 则按 PROPAGATION_REQUIRED 属性执行。

## 隔离级别
| 隔离级别 | 描述 | 可以避免的情况 |
| ------------ | ------------ | ------------ |
| ISOLATION_DEFAULT | 使用数据库默认的隔离级别 | 视数据库的隔离级别而定 |
| ISOLATION_READ_UNCOMMITTED | 最低的隔离级别，允许读取尚未提交的数据变更 | |
| ISOLATION_READ_COMMITTED | 允许读取并发事务已经提交的数据 | 脏读 |
| ISOLATION_REPEATABLE_READ | 对同一份数据多次读取结果都是一致的，除非数据是被本身事务自己所修改 | 脏读、不可重复读 |
| ISOLATION_SERIALIZABLE | 最高的隔离级别，所有的事务串行化执行，通常是通过完全锁定事务相关的数据库表来实现 | 脏读、不可重复读、幻读 |

## 只读事务
只读事务并不是一个强制选项，它只是一个“暗示”，提示数据库驱动程序和数据库服务，这个事务并不包含更改数据的操作，那么 JDBC 驱动和数据库就有可能根据这种情况对该事务进行一些特定的优化，比方说不安排相应的数据库锁，以减轻事务对数据库的压力。如果非要在只读事务里面修改数据，也并非不可以，只不过对于数据一致性的保护不像读写事务那样保险而已。 

## 超时时间
一个事务不能运行太长的时间，因为事务可能会对数据库进行加锁操作，所以长时间的事务会占用数据库资源。给事务设置超时时间，如果事务在超时时间内没有执行完毕，就会自动回滚，而不是一直等待事务结束。

# 事务状态
TransactionStatus 接口中提供了一些控制事务执行以及查看事务状态的方法。

```java
public interface TransactionStatus extends SavepointManager, Flushable {
	/**
	 * 是否是新的事务
	 */
	boolean isNewTransaction();
	/**
	 * 是否有恢复点
	 */
	boolean hasSavepoint();
	/**
	 * 设置为只回滚
	 */
	void setRollbackOnly();
	/**
	 * 是否为只回滚
	 */
	boolean isRollbackOnly();
	/**
	 * 是否已经完成
	 */
	boolean isCompleted();
}
```

# 参考
> [Spring 事务机制详解](https://www.open-open.com/lib/view/open1350865116821.html)

> [Spring 事务管理（详解和实例）](https://blog.csdn.net/trigl/article/details/50968079)

> [Spring 事务详细解释，满满的都是干货！](https://blog.csdn.net/u010963948/article/details/82761383)