---
title: 浅谈 JDBC
date: 2020/6/2 15:18:0
tags: [Java,数据库]
categories: [数据库]
---

应用程序与数据库软件进行交互可以有多种方式，其中常见的就是 JDBC 和 ODBC，它们都是在 X/Open SQL CLI（Call Level Interface）的基础上完成，但是 JDBC 更适合在 Java 应用中使用。JDBC 提供了一套用于 Java 程序与 DBMS 进行交互的 API，即只是制定了这个规范，具体的实现则交给了不同的数据库软件厂商。

<!--more-->

# JDBC 1.0
JDBC 的第一个版本是跟随 JDK 1.1 一起发布的，相关的接口和类位于 java.sql 包中，包含了 DriverManager 类、Driver 接口、Connection 接口、Statement 接口、ResultSet 接口、SQLException 类等。在这一时期，我们需要手动加载 JDBC 驱动，然后通过 DriverManager 来获取数据库连接，比较典型的使用方式如下：

```java
public class Test {
    public static void main(String[] args) {
        Connection connection = null;
        PreparedStatement preparedStatement = null;
        ResultSet resultSet = null;
        try {
            // 1 加载数据库驱动
            Class.forName("com.mysql.jdbc.Driver");
            // 2 通过驱动管理类获取数据库连接
            connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/test?characterEncoding=utf-8",
                    "root", "root");
            // 3 定义 sql 语句，? 表示占位符
            String sql = "select * from user where username = ?";
            // 4 获取预处理 statement
            preparedStatement = connection.prepareStatement(sql);
            // 5 设置参数，第一个参数为 sql 语句中参数的序号（从 1 开始），第二个参数为设置的参数值
            preparedStatement.setString(1, "saber");
            // 6 向数据库发出 sql 执行查询，查询出结果集
            resultSet = preparedStatement.executeQuery();
            // 7 遍历查询结果集
            while (resultSet.next()) {
                System.out.println(resultSet.getString("id"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 8 释放资源
            if (resultSet != null) {
                try {
                    resultSet.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
            if (preparedStatement != null) {
                try {
                    preparedStatement.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
            if (connection != null) {
                try {
                    connection.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

# JDBC 2.0
JDBC 2.0 的最大变化大概就是引入了 `javax.sql.DataSource`。在 JDBC 1.0 的时候，我们只能通过 DriverManager 来创建数据库连接，而在 JDBC 2.0 中，我们可以使用 DataSource 直接获取数据库连接。一个 DataSource 对象代表一个真正的数据源，在使用 DataSource 时，我们可以不必编写加载驱动的代码，只需要将数据库连接信息作为参数传入即可，DataSource 会负责连接数据库的工作。同时应用程序可以直接连接到多个数据源，我们只需要修改 DataSource 的属性即可，不需要修改应用程序的代码，与 DriverManager 相比，我们的代码会变的更加简洁，也更容易控制。

DataSource 大致有三种类型的实现，其中一种是最基本的实现，即生成标准的 Connection 对象。另一种是基于连接池的实现，即在 DataSource 中集成了连接池技术，生成的 Connection 对象是自动参与连接池的。还有一种就是分布式事务的实现，在这种实现中，一般会使用连接池和事务管理器，生成的 Connection 对象可以用于分布式事务。

DataSource 可以与 JNDI 一起工作，这样就可以把 DataSource 对象的创建、部署和管理与应用程序分开了，提高了应用程序的可移植性和可维护性。系统管理员或者具有相应权限的人来配置 DataSource 对象，包括设定 DataSource 的属性，然后将它注册到 JNDI 服务中去。在注册 DataSource 对象的的过程中，系统管理员需要把 DataSource 对象和一个逻辑名称关联起来。名称可以是任意的，通常选择能够代表数据源并且容易记忆的名称。

我们可以在 Java Web 项目的 META-INF 目录下新建一个 context.xml 文件，然后添加以下内容，然后修改 WEB-INF/web.xml 配置文件。或者直接在 Tomcat 中修改 conf/context.xml 文件，添加一个 Resource。直接修改 Tomcat 配置文件的这种方式，需要将用到的 jar 包拷贝到 Tomcat 安装目录下的 lib 目录中。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context>
  <Resource
    name="jdbc/druid-test"
    factory="com.alibaba.druid.pool.DruidDataSourceFactory"
    auth="Container"
    type="javax.sql.DataSource"
    maxActive="100"
    maxIdle="30"
    maxWait="10000"
    username="root"
    password="root"
    url="jdbc:mysql://localhost:3306/test?useSSL=false" />
</Context>
```

```java
public class Test {
    public static void main(String[] args) {
        try {
            Context context = new InitialContext();
            DataSource dataSource = (DataSource) context.lookup("java:comp/env/jdbc/druid-test");
            try {
                Connection connection = dataSource.getConnection();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        } catch (NamingException e) {
            e.printStackTrace();
        }
    }
}
```

# JDBC 4.0
JDBC 4.0 随着 JDK 6 一起发布，值得一说的是，在 JDK 6 中，使用 DriverManager 获取连接之前可以不再显式地加载驱动。`java.sql.DriverManager` 的内部实现机制决定了只有先通过 Class.forName() 方法找到特定驱动的 class 文件，DriverManager.getConnection() 方法才能顺利地获得数据库连接。这样的代码为编写程序增加了不必要的负担，JDK 的开发者也意识到了这一点。从 JDK 6 开始，应用程序不再需要显式地加载驱动程序了，DriverManager 开始能够自动地承担这项任务。这主要得益于 Java 的 SPI 机制。JDBC 4.0 的规范规定，所有的驱动 jar 包必须在 META-INF/services 目录下包含一个名称为 java.sql.Driver 的文本文件，这个文件里的一行便对应一个驱动类。有了这样的描述，DriverManager 就可以从当前的 CLASSPATH 中找到对应的驱动文件。