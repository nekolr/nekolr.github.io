---
title: Shiro RESTful 化改造
date: 2019/1/29 13:58:0
tags: [Shiro]
categories: [Shiro]
---

实际上 Shiro 是支持 RESTful 风格的权限校验的，默认提供的 `HttpMethodPermissionFilter` 过滤器可以将 HTTP 请求的方法转化为相应的动词，并使用该动词构造对应的权限。  

<!--more-->  

在使用该过滤器时，默认情况下一个 `/api/user/123` 的 GET 请求将被映射为 `user:read` 权限，然后通过 `Subject.isPermitted("user:read")` 来校验用户是否具有该权限。而一个 `/api/user` 的 POST 请求将被映射为 `user:create` 权限来进行校验。  

这里有一个 ini 配置文件，在 urls 中配置使用了 RESTful 风格的过滤器。  

```
[main]

[users]
# 用户名 = 密码, 角色1, 角色2,...
saber = saber, employee

[roles]
# 角色 = 权限1, 权限2,...
admin = *
employee = user:read:*

[urls]
# 所有的 /user/** 请求都经过 RESTful 风格的过滤器，映射成对应的权限后校验
/user/** = rest[user]
```

使用上述配置，在 saber 用户登录后，通过 GET 方法访问 `/user/123` 是可以通过的；而使用其他方法访问 `/user` 则会返回错误码 401。  

其实总的来看，Shiro 默认的过滤器已经能够满足基本的需求了，但是缺点是不够灵活，所以对其进行改造。在改造之前，先简单研究一下 Shiro 的设计。  

# Shiro 架构
![Shiro 架构](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/rzG.png)  

- **Subject**  
主体，代表当前用户。这个主体不一定是一个人，和当前应用交互的都是主体。在使用 Shiro 的过程中更多的是与 Subject 交互，所有的 Subject 都绑定到 Security Manager 中，与 Subject 的交互最终都会由 Security Manager 来执行。  
- **SecurityManager**  
安全管理器，是 Shiro 架构的核心，协调绑定在其上的所有的 Subject 和其它组件的工作。有点类似 Spring MVC 的 DispatcherServlet 前端控制器。  
- **Authenticator**  
身份认证器，负责用户登录的一个组件。当用户尝试登录时，Authenticator 知道如何协调一个或多个存储用户相关信息的 Realm（域），从这些 Realms 获取数据来验证用户身份。如果默认的 Authenticator 不能满足需求则可以自定义实现。如果配置了多个 Realm，则需要设置**认证策略（Authentication Strategy）**，即何种情况算作认证通过，如：必须所有的域都成功，只有一个成功等等。  
- **Authorizer**  
授权器，负责用户权限控制的一个组件。和 Authenticator 一样，Authorizer 知道如何协调一个或多个后端数据源来访问角色和权限信息，Authorizer 根据这些信息来确定用户是否具有该权限。  
- **SessionManager**  
会话管理器，负责创建和管理用户会话的生命周期。默认情况下，Shiro 将使用现有的会话机制，比如 Servlet 容器默认的会话机制，如果没有提供 Web/Servlet 或 EJB 容器，Shiro 也可以使用其内置的会话管理。  
- **SessionDAO**  
Session 持久化，使用它可以进行 Session 的 CRUD，可以将 Session 持久化到关系型或非关系型数据库中，我们只要实现对应的 SessionDAO 即可，同时还可以使用缓存来提高效率。  
- **Realm**  
域，允许有一个或多个域。可以把它看做是连接 Shiro 与安全数据（用户、角色、权限等）之间的“桥”或“连接器”，它封装了连接数据源的细节，Shiro 通过它们来获取安全数据，这些安全数据源可以是关系型数据库（JDBC）、LDAP、文本配置（INI 或属性文件等）等。**Shiro 不会去维护用户角色、权限，需要我们提供实现后通过相应的接口注入给 Shiro**。  
- **CacheManager**  
缓存管理，由于 Shiro 抽象了 Realm 这个概念，类似数据源，因此可以通过缓存这些域来提高效率，几乎所有的现代开源或企业级缓存产品都能够使用到 Shiro 中来。  
- **Cryptography**  
密码模块，提供了简单易用的加密和解密组件。  

# 入口分析
常见的使用 Shiro 的应用有三种。第一种就是普通的应用程序，不使用 Web 容器。第二种就是使用 Web 容器提供服务的应用，通过 Shiro 的过滤器实现用户的身份认证、权限校验等。最后一种就是在第二种的基础上集成了 Spring 容器。  

首先观察第一种应用使用 Shiro 的例子。  

```java
public class App {

    private static Logger log = LoggerFactory.getLogger(App.class);

    public static void main(String[] args) {
        // 创建 SecurityManager 工厂类，加载 ini 配置文件
        Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro.ini");
        // 获取 SecurityManager 实例
        SecurityManager securityManager = factory.getInstance();
        // 设置一次即可
        SecurityUtils.setSecurityManager(securityManager);
        // 获取 Subject
        Subject subject = SecurityUtils.getSubject();
        // 创建 Token
        UsernamePasswordToken token = new UsernamePasswordToken("saber", "saber");

        if (!subject.isAuthenticated()) {
            try {
                // 执行登录
                subject.login(token);
            } catch (UnknownAccountException uae) {
                // 账号不存在，给用户返回一个提示
                log.info("账号：" + token.getPrincipal() + "不存在");
                return;
            } catch (IncorrectCredentialsException ice) {
                // 错误的密码，给用户返回一个提示
                log.info("账号：" + token.getPrincipal() + "密码错误");
                return;
            } catch (LockedAccountException lae) {
                // 账号被锁定，给用户返回一个提示
                log.info("账号：" + token.getPrincipal() + "被锁定");
                return;
            } catch (AuthenticationException e) {
                log.info("其他错误");
                return;
            }
        }
        log.info("账号：" + subject.getPrincipal() + " 登录成功");

        /**
         * Session 测试
         */
        Session session = subject.getSession();
        session.setAttribute("someKey", "aValue");
        String value = (String) session.getAttribute("someKey");
        if (value.equals("aValue")) {
            log.info("Session 值正确");
        }

        /**
         * 角色测试
         */
        if (subject.hasRole("admin")) {
            log.info("具有 admin 角色");
        } else {
            log.info("不具有 admin 角色");
        }

        /**
         * 权限测试
         */
        if (subject.isPermitted("user:read")) {
            log.info("具有查看的权限");
        } else {
            log.info("不具有查看的权限");
        }
    }
}
```

在需要使用 Web 容器的应用中，一般会在 `web.xml` 中配置监听器和过滤器。我们知道，在 Servlet 容器启动的过程中，会先调用 Listener 的初始化方法，然后调用 Filter 的初始化方法。  

```xml
<listener>
    <listener-class>org.apache.shiro.web.env.EnvironmentLoaderListener</listener-class>
</listener>

<context-param>
    <param-name>shiroEnvironmentClass</param-name>
    <param-value>org.apache.shiro.web.env.IniWebEnvironment</param-value>
</context-param>

<context-param>
    <param-name>shiroConfigLocations</param-name>
    <param-value>classpath:shiro-web.ini</param-value>
</context-param>

<filter>
    <filter-name>shiroFilter</filter-name>
    <filter-class>org.apache.shiro.web.servlet.ShiroFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>shiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>FORWARD</dispatcher>
    <dispatcher>INCLUDE</dispatcher>
    <dispatcher>ERROR</dispatcher>
</filter-mapping>
```

其中，在调用监听器的初始化方法时会创建一个 `WebEnvironment` 放入 `ServletContext` 的全局属性中，它封装了 Shiro 运行时所需要的所有对象，比如 `SecurityManager`、`FilterChainResolver` 等。在 Web 容器启动时，`ShiroFilter` 的初始化方法中会从 Web 容器上下文的全局属性中获取 `WebEnvironment`，并将 `SecurityManager` 等值注入到过滤器中备用。  

在集成了 Spring 之后，就需要将 Shiro 纳入容器管理。Spring 提供了 `DelegatingFilterProxy` 类，它的作用就是代理 Filter。简单来说，它就是一个过滤器，因为它实现了 Filter 接口。在 web.xml 中配置它之后，就可以在 Web 容器启动时执行它的初始化方法，在初始化方法中会从 Spring 容器中寻找它代理的过滤器，然后注入到 `delegate` 属性中。同时它实现了 `DisposableBean` 接口，因此在需要销毁时，会调用 destroy 方法完成销毁。这样 Spring 就可以通过它间接管理过滤器的生命周期。  

```xml
<filter>
    <filter-name>shiroFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    <init-param>
        <param-name>targetFilterLifecycle</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>shiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

在 applicationContext.xml 中配置的 shiroFilter 是 `ShiroFilterFactoryBean`。我们知道，FactoryBean 是一种特殊的 bean，通过 getBean 方法从容器中获取实例时，其实获取到的是该实例的 getObject 方法返回的对象。  

```xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
    <property name="securityManager" ref="securityManager"/>
</bean>

<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
    <property name="realm" ref="myRealm"/>
</bean>

<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>
```

在查看了 `ShiroFilterFactoryBean` 后会发现，它的 getObject 方法里会创建 `PathMatchingFilterChainResolver`，而它并不能满足我们的需求，因此都需要重写。首先重写 `PathMatchingFilterChainResolver` 的 getChain 方法，我们约定配置的过滤链的 path 应包含请求路径和请求方法，格式为 `uri==httpMethod`，比如 `/user/nekolr==GET`。只有请求路径和请求方法同时匹配，才会认为真正匹配，并将请求交由过滤器处理。  

```java
public class RestPathMatchingFilterChainResolver extends PathMatchingFilterChainResolver {

    @Override
    public FilterChain getChain(ServletRequest request, ServletResponse response, FilterChain originalChain) {
        FilterChainManager filterChainManager = getFilterChainManager();
        if (!filterChainManager.hasChains()) {
            return null;
        }
        String requestURI = getPathWithinApplication(request);
        boolean isMatch;
        Iterator<String> iterator = filterChainManager.getChainNames().iterator();
        String pathPattern;
        String[] strings;
        do {
            if (!iterator.hasNext()) {
                return null;
            }
            pathPattern = iterator.next();
            strings = pathPattern.split("==");
            if (strings.length == 2) {
                // 比较 HTTP METHOD 是否一致，不一致就不匹配
                if (WebUtils.toHttp(request).getMethod().toUpperCase().equals(strings[1].toUpperCase())) {
                    isMatch = true;
                } else {
                    isMatch = false;
                }
            } else {
                isMatch = false;
            }
            // 重新赋值
            pathPattern = strings[0];
        } while (!this.pathMatches(pathPattern, requestURI) || !isMatch);

        if (log.isTraceEnabled()) {
            log.trace("Matched path pattern [" + pathPattern + "] for requestURI [" + requestURI + "].  " +
                    "Utilizing corresponding filter chain...");
        }
        if (strings.length == 2) {
            pathPattern = pathPattern.concat("==").concat(WebUtils.toHttp(request).getMethod().toUpperCase());
        }
        return filterChainManager.proxy(originalChain, pathPattern);
    }
}
```

```java
public class RestShiroFilterFactoryBean extends ShiroFilterFactoryBean {

    @Override
    protected AbstractShiroFilter createInstance() throws Exception {
        log.debug("Creating Shiro Filter instance.");
        SecurityManager securityManager = this.getSecurityManager();
        String msg;
        if (securityManager == null) {
            msg = "SecurityManager property must be set.";
            throw new BeanInitializationException(msg);
        } else if (!(securityManager instanceof WebSecurityManager)) {
            msg = "The security manager does not implement the WebSecurityManager interface.";
            throw new BeanInitializationException(msg);
        } else {
            FilterChainManager manager = this.createFilterChainManager();
            // RestPathMatchingFilterChainResolver
            RestPathMatchingFilterChainResolver chainResolver = new RestPathMatchingFilterChainResolver();
            chainResolver.setFilterChainManager(manager);
            return new RestShiroFilterFactoryBean.SpringShiroFilter((WebSecurityManager) securityManager, chainResolver);
        }
    }

    private static final class SpringShiroFilter extends AbstractShiroFilter {
        protected SpringShiroFilter(WebSecurityManager webSecurityManager, FilterChainResolver resolver) {
            if (webSecurityManager == null) {
                throw new IllegalArgumentException("WebSecurityManager property cannot be null.");
            } else {
                this.setSecurityManager(webSecurityManager);
                if (resolver != null) {
                    this.setFilterChainResolver(resolver);
                }

            }
        }
    }
}
```

