---
title: CAS 初探
date: 2021/3/29 16:34:0
tags: [CAS]
categories: [安全]
---

CAS（中央身份认证服务，Central Authentication Service）是由耶鲁大学发起的一个企业级开源项目，目的是帮助 Web 应用系统构建一种可靠的单点登录解决方案。

<!--more-->

# 几个概念 
CAS 主要包含两大组件：CAS Server 和 CAS Client，它们之间可以通过各种协议进行通信。在使用 CAS 之前，除了 CAS Server 和 CAS Client，还需要了解 TGT、TGC 以及 ST。

![CAS 的架构](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202104011547/2021/03/31/qNO.png)

## CAS Server
CAS Server 的主要职责就是通过签发和验证 Ticket 来认证用户并授予其对 CAS Client 的访问权限。当用户成功登录后，CAS Server 会给用户签发一个 Ticket，同时还会创建一个 SSO 会话（该 Ticket 在这里被称为 TGT，TGT 会以 Cookie 的形式保存在浏览器中）。当用户访问某个服务时，CAS Server 会根据用户提供的 TGT 来签发一个 Ticket 作为令牌（该 Ticket 在这里被称为 ST），并通过浏览器重定向的形式返回。此时，该服务会拿着重定向 URL 中携带的 Ticket，向 CAS Server 验证有效性，以判断是否放行。

CAS Server 是通过 Spring Framework 搭建的一个 Java Servlet 应用，在较新的版本中，则使用了 Spring Boot 框架，因此可以直接使用内置容器。CAS Server 一般都是独立部署的。可以通过[该项目](https://github.com/apereo/cas-overlay-template)快速部署一个 CAS Server。

## CAS Client
CAS Client 的主要职责就是保护 CAS 应用（可以理解为各种应用和服务，只不过它们都通过各种方式包装成了 CAS Client），并通过各种协议与 CAS Server 通信，以验证用户是否具有访问权限。

CAS Client 支持的软件平台和产品有很多，比如 Java、.NET、PHP 等等，不过与 Java 平台相比，其他平台的活跃性一般。

## TGT
TGT 全称为 Ticket-Granting Ticket，当用户登录成功后，用户的基本信息，登录有效期等信息都存储在这里。我们可以先简单地把用户登录后 CAS Server 为用户创建的 SSO 会话理解为我们之前使用的 HttpSession。在以前，我们的 Session 中会保存用户的一些信息，比如用户名。TGT 就是这些信息的一个封装。

## TGC
TGC 全称为 Ticket-Granting Cookie，TGT 会以 Cookie 的形式保存在浏览器中，用来维持与用户的会话。

## ST
ST 全称为 Service Ticket，它是 CAS Sever 通过 TGT 给用户发放的一张 Ticket，用户在访问其他服务时，如果发现没有 Cookie 或者 ST ，那么就会 302 到 CAS Server 去获取 ST，然后再携带着 ST 回来，接着 CAS Client 会通过 ST 去 CAS Server 上获取用户的登录状态。

# 一般流程
下面通过官方给出的流程图来看下 CAS 单点登录的一般流程是怎样的。

![CAS 登录授权的一般过程](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202104011547/2021/03/31/4w6.png)

首先用户通过浏览器访问受保护的 APP1，APP1 发现用户没有登录，所以返回 302，让用户到 CAS Server 的地址去登录，同时地址中还会包含一个 service 参数，对应的是 APP1 的地址。

接着浏览器会自动重定向到 CAS Server 的地址，由于用户没有与 CAS Server 建立 SSO 会话，所以用户需要登录。当用户登录成功后，CAS Server 会要求浏览器设置 TGC 并返回 302，让用户访问之前 service 参数指定的地址，同时地址中还会包含一个 ticket 参数，对应的就是 ST。

接着浏览器自动重定向到 APP1 的地址，并带有 ticket 参数。APP1 检测到 ticket 参数，会带着该参数去 CAS Server 验证 ticket 的有效性，验证成功后，APP1 会要求浏览器设置 session cookie 并返回 302，让用户重定向到 APP1 的地址（该地址不带 ticket 参数，目的是让浏览器地址栏隐藏 ST）。此时浏览器自动重定向，并带着 session cookie 访问 APP1，APP1 只需要验证 session cookie 即可确认是否放行。如果用户再次访问 APP1，只要 APP1 的 session cookie 还在，用户就可以一直访问 APP1。

当用户通过浏览器访问 APP2 时，同样 APP2 会返回 302，让用户到 CAS Server 去登录，地址中还会包含一个 service 参数，对应的是 APP2 的地址。由于用户的浏览器中带有 CAS Server 的 cookie（TGC），所以 CAS Server 会验证 TGT 的有效性，验证成功后会返回 302，让用户访问之前 service 参数指定的地址，同时地址中会包含一个 ticket 参数，对应的是 ST。接下来重复上述步骤，用户不需要再次登录就可以访问 APP2。

# CAS Server 部署
CAS Server 一般不需要从头编写并部署，使用官方提供的一个起始项目 [cas-overlay-template](https://github.com/apereo/cas-overlay-template)，通过少量的代码修改就可以快速部署。

> 在写这篇文章时，CAS 的最新稳定版为 v6.3.3，它要求 JDK 的版本为 JDK 11，由于 Oracle JDK 协议的修改，我们最好安装一个 OpenJDK 11。

首先将项目克隆下来，需要注意的是，该项目通过 Gradle 来构建，由于众所周知的原因，首先修改 `build.gradle` 配置文件中的 maven 地址为阿里云的镜像仓库地址 `maven { url "http://maven.aliyun.com/nexus/content/groups/public/" }`。

接着我们需要通过 `./gradlew.bat clean build` 命令来清理并构建项目，第一次构建的时候比较慢。在构建完成后，会在项目中生成一个 build 目录，其中 libs 目录下的 cas.war 就是整个 CAS Server 的部署文件。接着我们使用 `./gradlew.bat explodeWar` 命令可以将该 war 文件解压，解压后的文件位于 build 目录下的 cas-resources 目录，此时我们需要在 src/main 目录下新建一个 resources 目录，并将 cas-resources 目录下的文件复制到该目录下。

CAS Server 从版本 4 开始，要求使用 HTTPS 进行通信，所以我们得提前准备 HTTPS 证书。如果是公司里的项目，可能需要购买 HTTPS 证书，自己玩的话可以申请一个免费的 HTTPS 证书，或者使用 [mkcert](https://github.com/FiloSottile/mkcert) 来生成一个本地的 HTTPS 证书。

> 在本地开发中，我们经常需要模拟 HTTPS 环境，比如 PWA 应用就要求必须使用 https 访问。在传统的解决方案中，我们需要使用自签证书，然后在 http server 中使用自签证书。由于自签证书浏览器不信任，所以我们需要将自签证书使用的 CA 证书添加到系统或浏览器的可信 CA 证书中来规避这个问题。以前完成这些操作需要执行一系列繁琐的 openssl 命令，现在可以直接使用 mkcert 这个工具来简化这一过程，生成本地的 https 证书，并且信任自签 CA。

这里为了方便，我们就使用 mkcert 来生成一个本地的 HTTPS 证书。首先执行 `mkcert -install` 命令将 CA 证书添加到本地可信 CA 中，以后由该 CA 签发的证书在本地都是可信的。

接着使用 `mkcert -pkcs12 -p12-file filepath\localhost.p12 localhost 127.0.0.1` 命令生成一个 PKCS#12 文件格式的 HTTPS 证书（一般 PKCS#12 证书都需要一个加密口令，这里生成的默认口令是 changeit）。

最后我们需要通过 keytool 工具将该证书转换成 Java 特有的证书格式（KeyStore），以便 Tomcat 等容器识别。使用 `keytool -importkeystore -srckeystore 'filepath\localhost.p12' -destkeystore 'filepath\localhost.keystore'` 进行转换，转换过程中需要输入密码（changeit）。

我们将 keytool 转换后的证书文件放到 `src/main/resources` 目录下，然后修改 application.properties 文件内容：`server.ssl.key-store=classpath:localhost.keystore` 即可。

接下来我们可以使用 `./gradlew.bat run` 命令运行项目，也可以通过 `./gradlew.bat clean build` 命令构建新的 war 包后使用 `java -jar cas.war` 命令直接运行 war 包，然后通过地址：`https://localhost:8443/cas/login` 访问。

在不修改配置文件的情况下，默认的登录用户名和密码分别为：casuser 和 Mellon，这是通过 application.properties 文件配置的：`cas.authn.accept.users=casuser::Mellon`。一般后续会修改为通过数据库查询的方式验证，此时会注释掉该部分代码，并添加类似下面的代码：

```ini
##
# CAS Authentication Credentials
#
cas.authn.accept.enabled=false
#cas.authn.accept.users=casuser::Mellon
#cas.authn.accept.name=Static Credentials

##
# Common properties
# more info: https://apereo.github.io/cas/6.3.x/configuration/Configuration-Properties-Common.html#database-settings
#
cas.authn.jdbc.query[0].user=root
cas.authn.jdbc.query[0].password=root
cas.authn.jdbc.query[0].driver-class=com.mysql.jdbc.Driver
cas.authn.jdbc.query[0].url=jdbc:mysql://localhost:3306/cas?useUnicode=true&characterEncoding=utf-8&useSSL=false
cas.authn.jdbc.query[0].dialect=org.hibernate.dialect.MySQL57Dialect

##
# Password Encoding
# more info: https://apereo.github.io/cas/6.3.x/configuration/Configuration-Properties-Common.html#password-encoding
#

# @See PasswordEncoderProperties
# 使用 Bcrypt 算法，Spring Security's BCryptPasswordEncoder
cas.authn.jdbc.query[0].password-encoder.type=BCRYPT
# The encoding algorithm to use such as 'UTF-8'. Relevant when the type used is 'DEFAULT'
#cas.authn.jdbc.query[0].password-encoder.character-encoding=UTF-8
# The encoding algorithm to use such as 'MD5'. Relevant when the type used is 'DEFAULT' or 'GLIBC_CRYPT'.
#cas.authn.jdbc.query[0].password-encoder.encoding-algorithm=MD5
# Secret to use with STANDARD, PBKDF2, BCRYPT, GLIBC_CRYPT password encoders. Secret usually is an optional setting.
#cas.authn.jdbc.query[0].password-encoder.secret=
# 表示 hash 杂凑次数，可选值为 4 到 31，数值越高越安全，默认 10 次
cas.authn.jdbc.query[0].password-encoder.strength=10

##
# Query Database Authentication
# more info: https://apereo.github.io/cas/6.3.x/configuration/Configuration-Properties.html#query-database-authentication
#

# @See BaseJdbcAuthenticationProperties
#cas.authn.jdbc.query[0].credential-criteria=
# Name of the authentication handler
#cas.authn.jdbc.query[0].name=
# Order of the authentication handler in the chain
#cas.authn.jdbc.query[0].order=
# 用户登录时的查询语句
cas.authn.jdbc.query[0].sql=SELECT * FROM user WHERE username=?
# 匹配的密码属性列
cas.authn.jdbc.query[0].field-password=password
# 指定过期字段，tinyint(1) 类型，字段值为 1 表示过期，过期账号需要修改密码；值为 0 表示没有过期
#cas.authn.jdbc.query[0].field-expired=expired
# 指定禁用字段，tinyint(1) 类型，字段值为 1 表示禁用；值为 0 表示没有正常
#cas.authn.jdbc.query[0].field-disabled=disabled
# List of column names to fetch as user attributes
#cas.authn.jdbc.query[0].principal-attribute-list=uid,cn:commonName,age,id_card_num:idcardnum
```

由于开启 Database Authentication 需要依赖 `cas-server-support-jdbc` 库，因此修改 build.gradle 文件：

```groovy
dependencies {
    // Add modules in format compatible with overlay casModules property
    if (project.hasProperty("casModules")) {
        def dependencies = project.getProperty("casModules").split(",")
        dependencies.each {
            def projectsToAdd = rootProject.subprojects.findAll {project ->
                project.name == "cas-server-core-${it}" || project.name == "cas-server-support-${it}"
            }
            projectsToAdd.each {implementation it}
        }
    }
    // CAS dependencies/modules may be listed here statically...
    implementation "org.apereo.cas:cas-server-webapp-init:${casServerVersion}"
    // 由于该库依赖的 MySQL 驱动版本过高，所以需要排除掉后自行设置
    implementation ("org.apereo.cas:cas-server-support-jdbc:${casServerVersion}") {
        exclude group: "mysql", module: "mysql-connector-java"
    }
    implementation "mysql:mysql-connector-java:5.1.49"
}
```