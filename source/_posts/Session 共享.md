---
title: Session 共享
date: 2020/6/19 12:51:0
tags: [架构]
categories: [架构]
---

我们知道，Session 是由服务端生成的，在生成 Session 的同时还会生成一个唯一的 Session Id。一般的 Web 应用会在服务器内存当中开辟一部分内存用来存储 Session，同时这个 Session Id 会以 Cookie 的形式发送给浏览器，浏览器会将它保存到本地磁盘当中，在下次请求同一个域时会一并提交。然而当系统架构升级为分布式架构或者微服务架构时，作为流量入口的网关会通过负载均衡策略将用户的请求转发到不同的 Web 容器当中，这就可能会出现用户登录系统后，再次发送请求则被告知没有登录的情况。此时就需要某种机制来确保所有的 Web 容器能够共享用户的 Session 信息。

<!--more-->

# 不使用 Session
这不是随便说说，在某些场景下确实可以不使用 Session。现在的很多接口类系统中都提倡“无状态服务”，也就是说每一次的接口访问都不依赖于前一次的接口访问，这样服务端就可以很容易地进行水平扩展，也就不存在 Session 共享的问题了。比较典型的方案就是 JWT（JSON Web Tokens）。

**在传统的 Session-Cookie 机制中，状态是由服务端负责维护的；而在 JWT 中，状态是由客户端负责维护的**。JWT 的原理是，服务端在认证完成后，会将用户信息与一些 JWT 的配置信息相结合，最终通过某种算法生成一个很长的字符串。这个字符串由三部分组成，中间用 `.` 进行分隔，分别为头部（Header）、负载（Payload）和签名（Signature）。其中 Header 是一个 JSON 对象，存储的是签名使用的算法和最终生成的令牌的类型。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Payload 部分也是一个 JSON 对象，它用来存储实际需要传递的数据，JWT 规定了 7 个官方字段供我们选用：

```
iss (issuer)：签发人
exp (expiration time)：过期时间
sub (subject)：主题
aud (audience)：受众
nbf (Not Before)：生效时间
iat (Issued At)：签发时间
jti (JWT ID)：编号
```

Signature 部分是对前两部分的签名，这样可以有效防止数据被篡改。在生成签名时，需要指定一个密钥（secret），这个密钥需要在服务端进行保存，不能泄露给其他人。然后使用 Header 中指定的签名算法，按照下面的公式生成签名：

```
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

JWT 方案最大的问题就是无法在使用过程中将某个已经签发的令牌作废，或者更改令牌的权限。也就是说，一旦 JWT 签发了一个令牌，那么在到期之前，该令牌会始终有效。因此如果令牌发生泄漏我们也无法处理，只能等待令牌到期失效。造成这个问题的根本原因是我们将状态维护的工作交给了客户端，服务端已经无法对状态进行控制。网上有很多人为了解决这个问题，选择了引入类似 Redis 这种的存储系统来负责维护签发过的令牌，我不能说这种做法有多不好，毕竟这也能实现需求，但是这种做法明显已经偏离了 JWT 的初衷（stateless 无状态），还不如直接使用更加成熟的 Session-Cookie 机制。总的来说，JWT 应该被用在 Token 有效时间短，或者一次性使用的场景中。设置较短的有效时间可以降低 Token 泄露时的危险程度。

Cookie 的特性决定了它只能在浏览器中使用，现在的很多系统服务的设备类型多种多样，如果在移动设备上采用原生的方式开发 App（Native App），那么开发人员就需要自己手动模拟浏览器来维护 Cookie（手动解析 `Set-Cookie` Header 等），因此在一些系统中开发人员可能更倾向于使用 Token + Redis 的方案。用户登录后，服务端通过某种方式生成一个唯一的 Token 返回给客户端，客户端在接下来的每次请求的 Header 中都携带该 Token。而服务端则会将 Token 与用户信息以键值对的形式存储在 Redis 中，并设置过期时间。这种方案本质上还是实现了类似 Session-Cookie 的机制，它还是有状态的，只不过状态的维护由之前的 Web 容器转移到了集中式的内存服务（Redis）中。同时我们很容易地就能想到，这种将状态的维护转移并集中管理的做法同样可以应用到 Session-Cookie 机制当中。

# Session 同步
所谓 Session 同步就是通过网络将 Web 容器的 Session 复制到其他 Web 容器上，最终实现所有的 Web 容器都保存相同的 Session。这种方案的优点就是不需要修改代码，但是缺点也很多：首先需要 Web 容器支持 Session 同步，一旦容器不支持那就无法使用了。Tomcat 有一个开源的 Session 同步组件 [tomcat-redis-session-manager](https://github.com/jcoleman/tomcat-redis-session-manager)，但是它不支持 Tomcat 8 及以上的版本。同时如果数据量较大时同步还有可能会出现延迟，因此这个方案一般都不会考虑。

# IP 绑定
使用 Nginx（或其他负载均衡软硬件）中的 IP 绑定策略（`ip_hash` 策略），将同一个用户（相同 IP 地址）的请求都转发到固定的一个 Web 容器中，这么做同样可以避免修改代码，但是也失去了负载均衡的意义，当一台服务器宕机的时候，会影响一批用户的使用。

# 集中式存储
将 Session 交给一个集中式的存储系统负责维护，所有的 Web 容器都通过这个存储系统实现 Session 的存取。这种方案的优点就是可靠性较高，同时水平扩展方便，缺点是系统整体的复杂度提升，同时引入新的组件也就意味着可能会产生很多新的问题。这种方案是目前比较常用的方案，典型的开源实现就是 Spring Session。当然也可以选择自己实现，其核心思想就是重写 HttpServletRequestWrapper 中的 getSession 方法。

Spring Session 提供了多种存储方案的实现，比如 Redis、JDBC（关系型数据库）、MongoDB 等，我们这里使用 Spring Session Data Redis 模块来举一个例子。

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.1.RELEASE</version>
    <relativePath/>
</parent>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

使用 Spring Boot 可以减少很多手工配置的工作，接下来只需要使用 `@EnableRedisHttpSession` 注解就可以将 Session 的集中存储管理交给 Redis。

```java
@SpringBootApplication
@EnableRedisHttpSession
public class SpringSessionDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringSessionDemoApplication.class, args);
    }
}
```

在 `@EnableRedisHttpSession` 注解中，使用 `@Import` 注解引入了 RedisHttpSessionConfiguration 配置类，该配置类的主要目的是在容器中注册一个 SessionRepositoryFilter 过滤器，这个过滤器会拦截所有的请求，并将 HttpServletRequest 和 HttpServletResponse 包装成 SessionRepositoryRequestWrapper 和 SessionRepositoryResponseWrapper，在 Wrapper 类中重写了 getSession 等方法。