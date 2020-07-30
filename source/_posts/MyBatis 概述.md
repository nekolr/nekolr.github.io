---
title: MyBatis 概述
date: 2020/6/3 12:56:0
tags: [Java,MyBatis]
categories: [MyBatis]
---

好久不用 MyBatis，都快忘了怎么使用了，重新看一遍文档并简单记录下来，方便下次查阅。

<!--more-->

# 工作原理图
![工作原理图](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202006032025/2020/06/03/Bxg.png)

# 核心对象
`SqlSessionFactory` 是创建 SqlSession 的工厂类。SqlSessionFactory 对象的实例可以通过 SqlSessionFactoryBuilder 获得，具体来说就是通过 XML 配置文件或一个预先定制的 `org.apache.ibatis.session.Configuration` 实例构建出 SqlSessionFactory 的实例。SqlSessionFactory 是线程安全的，它一旦被创建，应该在应用执行期间都存在，同时需要注意不要重复创建，建议使用单例模式。

`SqlSession` 类似于 JDBC 中的 Connection，应用程序通过它与持久层进行交互。SqlSession 的底层封装了 JDBC 连接，可以用 SqlSession 实例来直接执行被映射的 SQL 语句。需要注意的是，SqlSession 不是线程安全的，因此每个线程都应该有它自己的 SqlSession 实例，要尽量避免 SqlSession 的实例被共享。

# 执行过程
首先读取配置文件（包括全局配置文件和映射文件），通过配置构造出 SqlSessionFactory，即会话工厂。接着通过 SqlSessionFactory 创建 SqlSession（即会话）。最后通过 SqlSession 来执行映射的 SQL 语句。SqlSession 本身是不能直接操作数据库的，它是通过底层的 Executor 执行器接口来操作数据库的。SqlSession 将要处理的 SQL 信息封装到一个底层对象 MappedStatement 中，该对象主要包括：SQL语句、输入参数映射信息、输出结果集映射信息等，然后再将封装的 MappedStatement 对象传入 Executor 中去执行相关操作。

```java
public class Application {

    public static void main(String[] args) throws IOException {
        // 读取配置文件
        InputStream input = Resources.getResourceAsStream("mybatis-config.xml");
        // 创建 SqlSessionFactory
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(input);
        // 创建 SqlSession
        SqlSession sqlSession = sqlSessionFactory.openSession();

        // 从 xml 映射文件中获取数据库操作语句
        User user = sqlSession.selectOne("com.github.nekolr.mapper.UserMapper.selectOne", 1);

        // 获取 Mapper 接口的代理对象
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        User other = userMapper.selectOne(2);

        // 用完关闭
        sqlSession.commit();
        sqlSession.close();
    }
}
```

Executor 接口的实现大致有两种，其中一种是普通的执行器，另外一种是缓存执行器（CachingExecutor）。CachingExecutor 有一个重要的属性就是 delegate，它保存的是某类普通的 Executor。在执行 update 操作时，它直接调用 delegate 的 update 方法，而在执行 query 方法时会先尝试从 cache 中取值，取不到再调用 delegate 的查询方法，并将查询结果存入 cache 中。

普通的 Executor 大致有三个：SimpleExecutor、ReuseExecutor 和 BatchExecutor。它们都继承自 BaseExecutor。SimpleExecutor 是一种常规执行器，每次执行都会创建一个 Statement，用完后关闭。ReuseExecutor 是可重用执行器，它会将 Statement 存入 map 中，这样就可以重用已经创建过的 Statement。BatchExecutor 是批处理执行器，doUpdate 预处理存储过程或批处理操作，doQuery 提交并执行过程。

# 接口绑定
我们定义的接口需要与 SQL 语句进行绑定之后才能使用。接口绑定有两种实现方式，一种是通过注解绑定，就是在接口的方法上添加 @Select、@Update 等注解，注解中包含 SQL 语句来进行绑定；另外一种就是通过 XML 里面编写 SQL 来进行绑定，在这种情况下，要指定 XML 映射文件里的 namespace 必须为接口的全限定类名。

# 映射器
MyBatis 的映射器大致包括 XML 映射，XML + 接口映射，注解 + 接口映射这三类。其中 XML 映射是最早提供和支持的，这种只需要定义映射的 xml 文件即可。

```java
// 传入映射文件的 namespace + 语句的 id
User user = sqlSession.selectOne("com.github.nekolr.mapper.UserMapper.selectOne", 1);
```

XML + 接口映射的方式需要在 XML 映射的基础上再添加一个接口，其中 XML 映射文件中的 namespace 应该对应接口的全限定类名。

```java
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
User user = userMapper.selectOne(1);
```

注解 + 接口映射的方式完全抛弃了 XML，这种方式将以前需要在 XML 映射文件中定义的 SQL 语句全部转移到了接口文件中，通过注解的方式在接口方法上添加 SQL 语句，需要注意的是，使用这种方式需要修改 MyBatis 的全局配置文件，将以前的 mapper 换成包名或者接口类名。

```java
public interface UserMapper {
    @Select("select * from user where id = #{id}")
    User selectOne(Integer id);
}
```

# 数据源
MyBatis 提供了三种内建的 DataSource，包括 UNPOOLED、POOLED 和 JNDI。其中 UNPOOLED 数据源没有使用连接池技术，因此每次请求都会新建和销毁连接。POOLED 就是使用了数据库连接池技术的数据源，而 JNDI 数据源是为了能在 EJB 或应用服务器这类容器中使用，容器可以集中或者在外部配置数据源，然后通过 JNDI 来引用。如果想使用第三方的数据库连接池，可以有两种实现方式。

其中一种是实现 `org.apache.ibatis.datasource.DataSourceFactory` 接口。

```java
public interface DataSourceFactory {

  void setProperties(Properties props);

  DataSource getDataSource();

}
```

另一种方式就是继承 `org.apache.ibatis.datasource.unpooled.UnpooledDataSourceFactory` 类。

```java
public class HikariDataSourceFactory extends UnpooledDataSourceFactory {

    public HikariDataSourceFactory() {
    }

    public HikariDataSourceFactory(String driver, String url, String username, String password) {
        HikariConfig config = new HikariConfig();
        config.setDriverClassName(driver);
        config.setJdbcUrl(url);
        config.setUsername(username);
        config.setPassword(password);
        this.dataSource = new HikariDataSource(config);
    }
}
```

为了让自定义的数据源生效，还需要修改 MyBatis 的全局配置文件。

```xml
<dataSource type="com.github.nekolr.HikariDataSourceFactory">
    <property name="driver" value="${jdbc.driver}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</dataSource>
```

# 事务
MyBatis 提供了两种类型的事务管理器，包括 JDBC 和 MANAGED。其中 JDBC 事务管理器使用的是 JDBC 的提交和回滚机制，它依赖从数据源获取的连接来管理事务的作用域。MANAGED 事务管理器则几乎什么都不做，它不会提交或者回滚一个连接，而是让容器（比如 Web 容器或者 Spring 容器）来管理事务的整个生命周期。如果我们的项目中使用了 Spring 和 MyBatis，那么就不需要给 MyBatis 配置事务管理器，因为 Spring 会用自带的事务管理器来覆盖当前的配置。

# ObjectFactory
ObjectFactory 是一个接口类，它默认的实现类是 DefaultObjectFactory。在 MyBatis 中，默认的 DefaultObjectFactory 要做的就是实例化查询结果对应的目标类。MyBatis 允许注册自定义的 ObjectFactory，只需要实现 ObjectFactory 接口并修改 MyBatis 的全局配置文件即可。在大多数情况下，我们都不需要自定义对象工厂，只需要继承 DefaultObjectFactory，然后通过一定的改写来完成我们所需要的工作。

# 缓存
MyBatis 的缓存分为一级缓存和二级缓存，其中一级缓存是会话级别的缓存，它的作用域是 SqlSession。在同一个会话中，查询会被缓存，但是一旦执行新增、修改和删除的操作，缓存就会被清空。

二级缓存是相对一级缓存而言的，二级缓存是 Mapper 级别的缓存，在同一个命名空间（namespace）下所有的操作都会影响着同一个缓存容器，即当多个 SqlSession 去执行同一个 Mapper 的操作时，它们的二级缓存是共享的。在 MyBatis 中，一级缓存默认是开启的，二级缓存默认是关闭的。

在一级缓存中，不同的 session 执行相同的 SQL 查询时，每次都需要查询数据库，这显然是一种浪费。另外，如果在项目中同时使用了 Spring 和 MyBatis，那么每次查询之后都要关闭 SqlSession，关闭之后数据就被清空了，所以此时一级缓存是没有意义的。这也是为什么需要二级缓存的一个原因。

开启二级缓存需要在 XML 映射文件中加入类似下面的配置（也可以在接口文件上使用 @CacheNamespace 注解）。其中 size 代表缓存容器可以存储的缓存对象的个数，eviction 代表使用何种算法来清除不需要的缓存，默认提供了四种缓存清除策略：LRU、FIFO、SOFT 和 WEAK，默认使用 LRU。flushInterval 代表缓存刷新的时间间隔，默认情况下不设置也就是没有刷新间隔，这样缓存仅仅会在调用相关语句时才进行刷新。readOnly 表示缓存是否会给所有的调用者返回相同的缓存实例，默认为 false。如果设置为 true，那么会直接返回缓存对象的实例，这些对象不能被修改，但是相对的，这种方式能够提供可观的性能提升。如果设置为 false，那么会返回缓存对象的拷贝，因此速度会更慢一些，但是也更安全。

```xml
<cache
  eviction="FIFO"
  flushInterval="60000"
  size="512"
  readOnly="true"/>
```

除了内置的缓存实现，我们还可以使用自定义的缓存实现，或者为其他第三方的缓存组件创建适配器，通过实现 `org.apache.ibatis.cache.Cache` 接口来完全覆盖缓存行为。

```xml
<cache type="com.github.nekolr.cache.MyCache" />
```

## 为什么不建议使用 MyBatis 缓存
**首先需要说一点，SQL 层面的缓存几乎没有价值，包括 MyBatis 的一级缓存和二级缓存，缓存需要基于业务来做。一个小知识点：MySQL 8.0 已经废弃了查询缓存。**

一级缓存也就是本地缓存，默认是 Session 级别的。一级缓存在大事务下可能会出现读到脏数据的情况，比如在会话 A 中先执行了查询，然后在会话 B 中执行了修改，会话 B 中的缓存被清空了，但是会话 A 的缓存还存在，此时在会话 A 中再次执行查询得到的还是旧数据，这种情况可以修改配置文件：`<property name="localCacheScope" value="STATEMENT" />`，将一级缓存的作用范围缩小到语句级别，这样一个查询语句在执行完毕后会清除缓存。

> 在 InnoDB 数据库中，如果需要确保一系列的语句操作要么全部成功，要么全部失败，可以通过显式指定 autocommit=0 的方式开启一个事务；否则默认每条语句都会隐式开启一个事务，并在语句执行完成后自动提交。在会话 A 中先后执行的两次查询对应的是两个事务，因此第二次查询理应查询到最新的数据（两次查询不在一个事务中）。

由于二级缓存是 namespace 级别的缓存，也就是说不同的 namespace 之间的缓存是相互隔离的。如果一个表的某些操作不在它独立的 namespace 下进行就有可能出现脏读的情况，比如在 UserMapper 中都是针对用户表的操作，但是在一个另一个 Mapper 中也有针对用户表的操作，如果在 UserMapper 中执行了新、修改或者删除操作，UserMapper 中的缓存会被清空，但是另一个 Mapper 中的缓存还有可能存在。比较容易出现脏读情况的就是多表操作，不管多表操作位于哪个 namespace 下，都会存在某个表不在该命名空间下的情况。当然这个情况可以使用 `cache-ref` 规避，使用它可以在多个命名空间中共享缓存配置和实例。

# 插件机制
MyBatis 允许我们在映射语句执行过程中的某一个点进行拦截调用。MyBatis 默认允许使用插件来拦截的方法调用包括：

```
Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
ParameterHandler (getParameterObject, setParameters)
ResultSetHandler (handleResultSets, handleOutputParameters)
StatementHandler (prepare, parameterize, batch, update, query)
```

如果我们想做的不仅仅是监控方法的调用，那么我们需要相当了解要重写的方法的行为，因为在试图修改或重写已有方法的行为时，很可能会破坏 MyBatis 的核心模块。这些都是更底层的类和方法，所以使用插件的时候要特别当心。使用插件是非常简单的，只需实现 Interceptor 接口，并指定想要拦截的方法签名即可。

```java
@Intercepts({@Signature(
        type = Executor.class,
        method = "update",
        args = {MappedStatement.class, Object.class})})
public class ExamplePlugin implements Interceptor {
    private Properties properties = new Properties();

    public Object intercept(Invocation invocation) throws Throwable {
        // implement pre processing if need
        Object returnObject = invocation.proceed();
        // implement post processing if need
        return returnObject;
    }

    public void setProperties(Properties properties) {
        this.properties = properties;
    }
}
```

```xml
<!-- mybatis-config.xml -->
<plugins>
  <plugin interceptor="com.github.nekolr.plugin.ExamplePlugin">
    <property name="someProperty" value="100"/>
  </plugin>
</plugins>
```

> MyBatis 的插件机制是通过 JDK 的动态代理 + 责任链设计模式实现的。

# 整合 Spring
首先需要加入 MyBatis-Spring 模块。

```xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis-spring</artifactId>
  <version>${mybatis.version}</version>
</dependency>
```

要和 Spring 一起使用 MyBatis，需要在 Spring 应用上下文中定义至少两样东西：一个 SqlSessionFactory 和至少一个数据映射器类。在 MyBatis-Spring 中，可以使用 SqlSessionFactoryBean 来创建 SqlSessionFactory。

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
</bean>
```

```java
@Bean
public SqlSessionFactory sqlSessionFactory() throws Exception {
  SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
  factoryBean.setDataSource(dataSource());
  return factoryBean.getObject();
}
```

然后可以通过 MapperFactoryBean 将映射接口添加到 Spring 中，这种方式一次只能配置一个数据映射器类。也可以使用 MapperScannerConfigurer 直接扫描所有的数据映射器。

```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
  <property name="mapperInterface" value="com.github.nekolr.mapper.UserMapper" />
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

```java
@Bean
public MapperFactoryBean<UserMapper> userMapper() throws Exception {
  MapperFactoryBean<UserMapper> factoryBean = new MapperFactoryBean<>(UserMapper.class);
  factoryBean.setSqlSessionFactory(sqlSessionFactory());
  return factoryBean;
}
```

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
  <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
  <!-- 扫描接口包 -->
  <property name="basePackage" value="com.github.nekolr.mapper" />
</bean>
```

或者更简单地，直接在一个 Spring 配置类上使用扫描注解即可。

```java
@Configuration
@MapperScan("com.github.nekolr.mapper")
public class AppConfig {
}
```

使用 MyBatis-Spring 的一个重要原因是它允许 MyBatis 参与到 Spring 的事务管理中，而不是给 MyBatis 创建一个新的专用事务管理器。MyBatis-Spring 借助了 Spring 中的 DataSourceTransactionManager 来实现事务管理。一旦配置好 Spring 的事务管理器，我们就可以在 Spring 中按照平时的方式来使用事务，它支持 @Transactional 注解和 AOP 风格的配置。在事务处理期间，一个单独的 SqlSession 对象将会被创建和使用。当事务完成时，这个 session 会以合适的方式提交或回滚。

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <constructor-arg ref="dataSource" />
</bean>
```

```java
@Bean
public DataSourceTransactionManager transactionManager() {
  return new DataSourceTransactionManager(dataSource());
}
```

# 参考
> [MyBatis 官网](https://mybatis.org/mybatis-3/)

> [MyBatis Blog](https://blog.mybatis.org/)