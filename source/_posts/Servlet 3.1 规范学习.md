---
title: Servlet 3.1 规范学习
date: 2018/4/27 16:7:0
tags: [Servlet,Java Web]
categories: [Servlet]
---

此次学习 Servlet 3.1 规范的目的是整理学习规范中新的特性（一些特性在 3.0 已经出现），不会涵盖规范的全部内容，仅为个别重点。  

<!--more-->

# 编程注册组件
除了使用 web.xml 以及注解来配置组件外，新的规范还支持以编程的方式动态注册组件（Servlet、Listener 以及 Filter），具体来说是在 Web 容器启动时动态注册。我们可以使用 Servlet API 提供的 addServlet()、addFilter() 以及 addListener() 等方法来动态注册这些组件。注册的切入点有两种选择，其中一种是实现 `javax.servlet.ServletContextListener` 接口。

```java
@WebListener
public class ServletContextListener implements javax.servlet.ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        LogUtil.log("---------------------ServletContextInitialized---------------------");
        ServletContext servletContext = servletContextEvent.getServletContext();
        ServletRegistration servletRegistration = servletContext.
                addServlet(IndexServlet.class.getSimpleName(), IndexServlet.class);
        servletRegistration.addMapping("/index");
    }
    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {
        LogUtil.log("---------------------ServletContextDestroyed---------------------");
    }
}
```

另外一种是实现 `javax.servlet.ServletContainerInitializer` 接口，采用这种方法能够在容器运行时通过可插拔的方式来注册组件。

# Web 模块部署描述符片段
从 Servlet 3.0 规范就开始提供一些注解来简化 web.xml 的配置，如：`@WebServlet`、`@WebListener`、`@WebFilter` 等等，这些注解的使用暂且不提。Servlet 3.0 新增可插拔性来增加 Servlet 配置的灵活性，引入了名为“Web 模块部署描述符片段”的 web-fragment.xml 部署文件。该文件必须放在 jar 文件的 META-INF 目录下，该部署描述符文件可以包含一切可以在 web.xml 文件中定义的内容。通过这种方式，能够将某些 Servlet 组件打包成 jar 文件，在需要时引入，不需要时卸载。

在新的规范下，我们为 Web 应用增加一个 Servlet（或 Filter、Listener 同理）也就有三种方式：

- 继承 `javax.servlet.http.HttpServlet`，修改 web.xml 文件，增加一个 Servlet 配置项。  
- 继承 `javax.servlet.http.HttpServlet`，并为该类加上 `@WebServlet` 注解。  
- 继承 `javax.servlet.http.HttpServlet`，将该类打成 jar 包，并在 jar 包的 META-INF 目录下添加 web-fragment.xml 文件来配置相应的 Servlet。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-fragment xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee 
         http://xmlns.jcp.org/xml/ns/javaee/web-fragment_3_1.xsd"
         version="3.1">

    <servlet>
        <servlet-name>index</servlet-name>
        <servlet-class>com.nekolr.servlet.IndexServlet</servlet-class>
    </servlet>

    <servlet-mapping>
        <servlet-name>index</servlet-name>
        <url-pattern>/index</url-pattern>
    </servlet-mapping>

</web-fragment>
```

在 `web-app` 元素中包含一个新的 `metadata-complete` 属性，该属性用于描述当前 web 描述符是否完整，即当前配置是否完全，默认情况下该属性值为 false。当该属性值为 true 时，表示当前配置完全，Web 容器只会应用该配置，忽略其他类文件的注解（如 @WebServlet）以及 web-fragement.xml 的配置；当该属性值为 false 时，容器会扫描其他类文件的注解以及 web-fragement.xml 的配置。

由于规范允许应用配置多个文件（一个 web.xml 以及多个 web-fragement.xml），从应用中的多个不同的位置发现和加载配置，因此加载的顺序必须处理。具体细节参考规范中的说明。  

# 运行时可插拔性
为了实现运行时可插拔，需要实现 `javax.servlet.ServletContainerInitializer` 接口。同时，我们的实现必须在 jar 包的 META-INF/services 目录中一个名为 `javax.servlet.ServletContainerInitializer` 的文件中指定。  

![javax.servlet.ServletContainerInitializer](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/27/O7n.png)

```java
public class WebInitializer implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
        ServletRegistration servletRegistration = servletContext.
                addServlet(IndexServlet.class.getSimpleName(), IndexServlet.class);
        servletRegistration.addMapping("/index");
    }
}
```

典型的例子是 Spring，查看 spring-web 的 META-INF/services，果然有名为 javax.servlet.ServletContainerInitializer 的文件。

![spring-web](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/27/a9N.png)

```
org.springframework.web.SpringServletContainerInitializer
```

```java
@HandlesTypes({WebApplicationInitializer.class})
public class SpringServletContainerInitializer implements ServletContainerInitializer {
  @Override
  public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
      throws ServletException {

    List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

    if (webAppInitializerClasses != null) {
      for (Class<?> waiClass : webAppInitializerClasses) {
        // Be defensive: Some servlet containers provide us with invalid classes,
        // no matter what @HandlesTypes says...
        if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
            WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
          try {
            initializers.add((WebApplicationInitializer) waiClass.newInstance());
          }
          catch (Throwable ex) {
            throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
          }
        }
      }
    }

    if (initializers.isEmpty()) {
      servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
      return;
    }

    servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
    AnnotationAwareOrderComparator.sort(initializers);
    for (WebApplicationInitializer initializer : initializers) {
      initializer.onStartup(servletContext);
    }
  }
}
```

在 SpringServletContainerInitializer 类上有一个 @HandlesTypes 注解，按照规范中的意思，这个注解可以用在一些 javax.servlet.ServletContainerInitializer 的实现类上，@HandlesTypes 注解的 value 值是一个 Class 类型的数组，通过 onStartup() 方法中的 webAppInitializerClasses 参数，我们可以获取这些类型。比如在 Spring 的实现中，SpringServletContainerInitializer 作为实现类，使用 @HandlesTypes 注解的 value 值为 WebApplicationInitializer 接口，并且通过它的 onStartup() 方法也能够发现，实际调用的是 WebApplicationInitializer 的 onStartup() 方法。我们可以模仿 Spring 的这种实现。

```java
@HandlesTypes(value = AppInitializer.class)
public class WebInitializer implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
        List<AppInitializer> initializers = new LinkedList<>();
        if (set != null) {
            for (Class<?> clazz : set) {
                if (!clazz.isInterface() && !Modifier.isAbstract(clazz.getModifiers()) &&
                        AppInitializer.class.isAssignableFrom(clazz)) {
                    try {
                        initializers.add((AppInitializer) clazz.newInstance());
                    } catch (InstantiationException e) {
                        e.printStackTrace();
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
            }

            if (initializers.isEmpty()) {
                servletContext.log("No AppInitializer types detected on classpath");
                return;
            }

            servletContext.log(initializers.size() + " AppInitializer detected on classpath");
            for (AppInitializer appInitializer : initializers) {
                appInitializer.onStart(servletContext);
            }
        }
    }
}
```

实际是使用的 `AppInitializer` 接口的 `onStartup()` 方法。  

```java
public interface AppInitializer {

    void onStart(ServletContext servletContext) throws ServletException;
}
```

```java
public class CustomAppInitializer implements AppInitializer {
    @Override
    public void onStart(ServletContext servletContext) throws ServletException {
        ServletRegistration servletRegistration = servletContext.
                addServlet(IndexServlet.class.getSimpleName(), IndexServlet.class);
        servletRegistration.addMapping("/index");
    }
}
```

# Session 与 Cookie

我们知道 HTTP 协议是无状态的，即没有记忆和存储的能力，这意味着当处理过一次的业务，再次请求时需要重传，这会导致每次传输的数据量很大，尤其是在客户端与服务器进行交互的 Web 应用程序出现后，这种特性严重阻碍了这些程序的实现，因为交互是需要承前启后的。为了解决这个问题，Session 和 Cookie 诞生了。

与 Cookie 相关的标准有四个，最早的是 Netscape 标准，这也是最原始的 Cookie 规范，同时也是 [RFC 2109](https://tools.ietf.org/html/rfc2109) 规范的基础。1997 年 2 月份，Network Working Group 推出了第一份官方的 Cookie 标准，理论上，所有使用 Cookie 的客户端与服务端都应该实现该标准，但是由于那时的浏览器市场还是 Netscape 的天下，导致很多服务端仍然在使用 Netscape 标准。为了解决这个问题，2000 年 Network Working Group 又推出了 [RFC 2965](https://tools.ietf.org/html/rfc2965) 规范，该规范中增加了两个新的 Header：Cookie2 和 Set-Cookie2，其他部分与 RFC 2109 相差不多。2011 年，时隔 11 年，新的 Cookie 标准：[RFC 6265](https://tools.ietf.org/html/rfc6265) 规范发布。新规范可以说是把 Cookie 规则整个翻新了一遍，修改幅度很大，值得一提的是，新规范中增加了 HttpOnly 属性，指定 HttpOnly 的 Cookie 不能被客户端读写，仅供 HTTP 传输使用，或者只有服务端可以读写，客户端需要确保其不能读写。

Cookie 由服务器端生成（客户端也可以自己创建），发送给 User-Agent（一般是浏览器），浏览器会将 Cookie 的 key/value 值保存到某个目录下的文本文件中，在下次请求同一个域时会在请求头中加上该 Cookie（前提是浏览器没有禁用 Cookie）。

> Cookie 可以设置 domain 属性，该属性可以控制能够访问该 Cookie 的域名。比如设置为 `.google.com`，那么所有以 `google.com` 结尾的域名都可以访问该 Cookie，也就意味着该 Cookie 会被发送给所有以 `google.com` 结尾的子域名。

Session 由服务器端程序生成，在生成 Session 时会生成一个唯一的 Session Id，一般的 Web 应用会将 Session 存储到服务器内存当中，同时这个 Session Id 会以 Cookie 的形式发送给浏览器，浏览器会将它保存到本地磁盘当中，在下次请求同一个域时一并提交。

## Java 中的 Session 操作

- 创建

需要注意的是，有很多人误认为在一开始访问 Web 容器时就创建 Session，之所以这么认为大概是觉得一次访问就是一次会话，就会创建 Session。其实创建 Session 需要程序使用 `HttpServletRequest.getSession(true)` 方法。当传入 true 值时，会先获取 Session 对象，如果不存在则新建；当传入 false 值时，只获取 Session 对象，如果不存在则返回 null。容器创建 Session 时会生成一个唯一的 Session Id（Tomcat 服务器生成 Session Id 的方式是随机数+时间+jvmid），并将该 Id 发送给客户端。  

- 删除

Session 的删除有两种方式，一种是 Session 超时，由容器自行删除；一种是手动调用 `HttpSession` 对象的 `invalidate()` 方法。**需要注意的是，Session 不会因为浏览器的关闭而删除，删除只能通过上述的两种方式**。  

下面使用代码例子来证实。先写一个监听器监听 Session 的变化。  

```java
@WebListener
public class SessionListener implements HttpSessionListener {
    @Override
    public void sessionCreated(HttpSessionEvent se) {
        LogUtil.log("创建了 Session：" + se.getSession().getId());
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        LogUtil.log("删除了 Session：" + se.getSession().getId());
    }
}
```

接下来是两个 Servlet，一个是登录 Servlet，一个是主页 Servlet。  


```java
@WebServlet(urlPatterns = "/login")
public class LoginServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        LogUtil.log("---------------------doGet---------------------");
        String user = req.getParameter("user");
        String password = req.getParameter("password");
        if (user != null && password != null) {
            HttpSession session = req.getSession(true);
            session.setMaxInactiveInterval(10); // 10s 过期
            session.setAttribute("user", user);
            req.getRequestDispatcher("/WEB-INF/index.jsp").forward(req, resp);
            return;
        } else {
            resp.sendRedirect("/login.html");
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        LogUtil.log("---------------------doPost---------------------");
        this.doGet(req, resp);
    }

    @Override
    public void init() throws ServletException {
        LogUtil.log("---------------------LoginServletInitialized---------------------");
    }
}
```

```java
@WebServlet(value = "/index")
public class IndexServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        LogUtil.log("---------------------doGet---------------------");
        HttpSession session = req.getSession(false);
        if (session != null) {
            req.getRequestDispatcher("/WEB-INF/index.jsp").forward(req, resp);
            return;
        } else {
            resp.sendRedirect("/login.html");
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        LogUtil.log("---------------------doPost---------------------");
        this.doGet(req, resp);
    }
}
```

两个页面，一个是登录页面，一个是主页。需要注意的是，这个例子是为了观察 Session 的处理是不是我们预期的方式，而 jsp 在编译后会自动加上创建 Session 的代码，一种方式是在 `<%@ page session="false"%>` 中显式表明不使用 Session，另一种方式就是直接使用 html。  


```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>login</title>
</head>
<body>
<form action="/login" method="post">
    <input type="text" name="user"/>
    <input type="password" name="password"/>
    <button type="submit"> 登录 </button>
</form>
</body>
</html>
```

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<head>
    <title>index</title>
</head>
<body>
<%="欢迎你，" + session.getAttribute("user")%>
</body>
</html>
```

# 参考链接

> [JSR 340 Java Servlet 3.1](https://jcp.org/aboutJava/communityprocess/final/jsr340/index.html)

> [Servlet 3.0 新特性详解 ](https://www.ibm.com/developerworks/cn/java/j-lo-servlet30/)