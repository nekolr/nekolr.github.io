---
title: 服务端 WebSocket API
date: 2018/1/6 15:52:0
tags: [WebSocket]
categories: [WebSocket]
---
抛开别的语言不谈，使用 Java 实现的 WebSocket API 由 Oracle 制定，并已经成为 Java EE 7 的一部分，我们先使用该 API 简单实现一个 WebSocket 通信 demo。		
<!--more-->
		
## 前置条件
JDK 需要大于等于 7、servlet-api 3.1、websocket-api 1.1 以及实现了 [JSR356](https://github.com/javaee/websocket-spec) 规范的 Web 容器。  

> Tomcat 7 及以上版本都可以，Tomcat 7 以前的版本提供了 Tomcat 自己的 WebSocketServlet，在 Tomcat 7 中 `deprecated`，在 Tomcat 8 中移除。  

## 基于 JSR356 WebSocket API 的 demo
页面的代码如下：  

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>index</title>
</head>
<body>
<div>
    <textarea id="txt" style="width: 210px;height: 110px;"></textarea><br>
    <button id="connect"> 新建连接</button>
    <button id="send"> 发送消息</button>
    <button id="close"> 断开连接</button>
</div>
<div id="info"></div>
</body>
<script type="text/javascript" src="/js/websocket.js"></script>
</html>
```

```js
/* websocket.js */
var $ = function (id) {
    return document.querySelector(id);
};

var ws = null;
var txt = $("#txt");
var connect = $("#connect");
var send = $("#send");
var close = $("#close");
var info = $("#info");

connect.addEventListener("click", function () {
    ws = new WebSocket("ws://localhost:8080/web/websocket");

    ws.addEventListener("open", function (open) {
        info.innerHTML += "<p>" + new Date() + " 连接建立 </p>";
    });

    ws.addEventListener("message", function (message) {
        info.innerHTML += "<p>" + new Date() + " 服务器返回消息：" + message.data + "</p>";
    });

    ws.addEventListener("error", function (error) {
        info.innerHTML += "<p>" + new Date() + " 出错 </p>";
    });

    ws.addEventListener("close", function (close) {
        info.innerHTML += "<p>" + new Date() + " 连接关闭：" + close.code + "</p>";
    });
});

send.addEventListener("click", function () {
    if (txt.value.length > 0 && ws) ws.send(txt.value);
});

close.addEventListener("click", function () {
    if (ws) {
        ws.close();
    }
});
```

后端 Java 代码如下：  

```java
package com.nekolr;

import javax.websocket.*;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;

@ServerEndpoint("/web/websocket")
public class AnnotationEndpoint {

    @OnOpen
    public void onOpen(Session session) {
        // 后续可以将 session 维护进内存中，并建立相关联的 key，根据这个 key 查找出对应的 session，并往该 session 输出
        System.out.println("------连接建立------");
    }

    @OnMessage
    public void onMessage(Session session, String text) {
        if(session.isOpen()) {
            try {
                session.getBasicRemote().sendText(text);
            } catch (IOException e) {
                e.printStackTrace();
                try {
                    session.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
            }
        }
    }

    @OnClose
    public void onClose(Session session, CloseReason closeReason) {
        System.out.println("------连接关闭------");
    }
}
```
		
除了使用注解的方式，也可以通过继承 Endpoint 来实现，在这之前，需要先写一个配置类。		

```java
package com.nekolr;

import javax.websocket.Endpoint;
import javax.websocket.server.ServerApplicationConfig;
import javax.websocket.server.ServerEndpointConfig;
import java.util.HashSet;
import java.util.Set;

public class WebSocketConfig implements ServerApplicationConfig {

    /**
     * 该方法在容器启动后会扫描实现 Endpoint 的类，传入 set
     * 因此可以在该方法中过滤，自定义哪些可以部署
     * @param set 扫描到的处理类
     * @return 可以部署的处理类
     */
    @Override
    public Set<ServerEndpointConfig> getEndpointConfigs(Set<Class<? extends Endpoint>> set) {
        Set<ServerEndpointConfig> result = new HashSet<>();
        for (Class<? extends Endpoint> ep : set) {
            result.add(ServerEndpointConfig.Builder.create(ep, "/web/websocket").build());
        }
        return result;
    }

    /**
     * 该方法在容器启动后会扫描带有 Endpoint 注解的类，传入 set
     * 因此可以在该方法中过滤，自定义哪些可以部署
     * @param set 扫描到的处理类
     * @return 可以部署的处理类
     */
    @Override
    public Set<Class<?>> getAnnotatedEndpointClasses(Set<Class<?>> set) {
        return set;
    }
}
```
		
```java
package com.nekolr;


import javax.websocket.*;
import java.io.IOException;

public class CommonEndpoint extends Endpoint {

    @Override
    public void onOpen(Session session, EndpointConfig endpointConfig) {
        System.out.println("------连接建立------");
        session.addMessageHandler(new NekoMessageHandler(session.getBasicRemote()));
    }

    @Override
    public void onClose(Session session, CloseReason closeReason) {
        System.out.println("------连接关闭------");
        super.onClose(session, closeReason);
    }

    private static class NekoMessageHandler implements MessageHandler.Whole<String> {

        private final RemoteEndpoint.Basic remoteEndpointBasic;

        private NekoMessageHandler(RemoteEndpoint.Basic remoteEndpointBasic) {
            this.remoteEndpointBasic = remoteEndpointBasic;
        }

        @Override
        public void onMessage(String message) {
            if (remoteEndpointBasic != null) {
                try {
                    remoteEndpointBasic.sendText(message);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

## Spring 实现
Spring 从 4.0 版本后开始支持 JSR356 WebSocket API，同时也提供了一套 Spring 的 WebSocket API。		
		
我们在使用 JSR356 WebSocket API 时，是通过 Servlet 容器扫描来部署 Endpoint 处理类的，现在我们使用 Spring 来处理这个过程。		
		
```java
package com.nekolr;

import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;

import javax.servlet.ServletRegistration;

/**
 * Web 启动初始化
 */
public class WebInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(javax.servlet.ServletContext servletContext) {
        AnnotationConfigWebApplicationContext ctx = new AnnotationConfigWebApplicationContext();
        // 注册配置类
        ctx.register(MvcConfig.class, NekoEndpointConfig.class);
        ctx.setServletContext(servletContext);
        // 添加 Servlet（相当于 web.xml 的 servlet 配置）
        ServletRegistration.Dynamic servlet = servletContext.addServlet("mvc", new DispatcherServlet(ctx));
        servlet.addMapping("/");
    }
}
```
		
```java
package com.nekolr;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerMapping;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;

@Configuration
@EnableWebMvc
@ComponentScan(value = {"com.nekolr"})
public class MvcConfig extends WebMvcConfigurationSupport {

    /**
     * 映射静态资源文件目录
     * @param registry
     */
    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/js/**").addResourceLocations("/js/");
        registry.addResourceHandler("/views/**").addResourceLocations("/WEB-INF/views/");
    }

    /**
     * 资源映射器
     * @return
     */
    @Bean
    public HandlerMapping getHandlerMapping(){
        return super.resourceHandlerMapping();
    }
}
```
		
```java
package com.nekolr;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;
import org.springframework.web.socket.server.standard.ServerEndpointRegistration;

@Configuration
public class NekoEndpointConfig {
    /**
     * 为了让 Spring 管理 Endpoint，需要注入 ServerEndpointExporter，
     * 该 bean 会自动注册使用了@ServerEndpoint 注解声明的 Endpoint
     * @return
     */
    @Bean
    public ServerEndpointExporter endpointExporter() {
        return new ServerEndpointExporter();
    }

    /**
     * 通过 ServerEndpointRegistration 发布 Endpoint
     * @return
     */
    @Bean
    public ServerEndpointRegistration registration() {
        return new ServerEndpointRegistration("/web/websocket", NekoEndpoint.class);
    }
}
```
		
```java
package com.nekolr;

import javax.websocket.*;
import java.io.IOException;

public class NekoEndpoint extends Endpoint {

    @Override
    public void onOpen(Session session, EndpointConfig endpointConfig) {
        session.addMessageHandler(new MessageHandler.Whole<String>() {

            @Override
            public void onMessage(String s) {
                try {
                    session.getBasicRemote().sendText(s);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

        });
    }

    @Override
    public void onClose(Session session, CloseReason closeReason) {
        super.onClose(session, closeReason);
    }
}
```
		
下面我们再使用 Spring 提供的一套 API 来实现，同时，我们在页面使用 SockJS，Spring 对 SockJS 有很好的支持。与 JSR356 WebSocket API 的 Endpoint 不同，使用 Spring WebSocket API，Endpoint 变成了 WebSocketHandler，同时也更容易理解了。  

```xml
<dependencies>
    <!-- servlet-api -->
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>3.1.0</version>
        <scope>provided</scope>
    </dependency>
    <!-- spring-context -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>4.3.23.RELEASE</version>
    </dependency>
    <!-- spring-web -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
        <version>4.3.23.RELEASE</version>
    </dependency>
    <!-- spring-webmvc -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>4.3.23.RELEASE</version>
    </dependency>
    <!-- spring-websocket -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-websocket</artifactId>
        <version>4.3.23.RELEASE</version>
    </dependency>
    <!-- 一定加上这个包 -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.8.6</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

```java
package com.nekolr;

import org.springframework.web.socket.*;

public class NekoWebSocketHandler implements WebSocketHandler {
    @Override
    public void afterConnectionEstablished(WebSocketSession webSocketSession) {
        System.out.println("---------连接建立------------");
    }

    @Override
    public void handleMessage(WebSocketSession webSocketSession, WebSocketMessage<?> webSocketMessage)
            throws Exception {
        TextMessage returnMessage = new TextMessage(webSocketMessage.getPayload().toString());
        System.out.println(webSocketSession.getHandshakeHeaders().getFirst("Cookie"));
        webSocketSession.sendMessage(returnMessage);
    }

    @Override
    public void handleTransportError(WebSocketSession webSocketSession, Throwable throwable)
            throws Exception {
        if(webSocketSession.isOpen()) {
            webSocketSession.close();
        }
        System.out.println("---------连接出错------------");
    }

    @Override
    public void afterConnectionClosed(WebSocketSession webSocketSession, CloseStatus closeStatus) {
        System.out.println("---------连接断开------------");
    }

    @Override
    public boolean supportsPartialMessages() {
        return false;
    }
}
```
		
同时还可以写拦截器来过滤握手请求。		
		
```java
package com.nekolr;

import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.support.HttpSessionHandshakeInterceptor;

import java.util.Map;

/**
 * 可以在此拦截器中实现限制某些握手请求
 */
public class NekoHandshakeInterceptor extends HttpSessionHandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) 
            throws Exception {
        if (request.getHeaders().containsKey("Sec-WebSocket-Extensions")) {
            request.getHeaders().set("Sec-WebSocket-Extensions", "permessage-deflate");
        }
        return super.beforeHandshake(request, response, wsHandler, attributes);
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, 
                               WebSocketHandler wsHandler, Exception ex) {
        super.afterHandshake(request, response, wsHandler, ex);
    }
}
```

同时还需要写一个配置类来部署这些处理器类。		
		
```java
package com.nekolr;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;
import org.springframework.web.socket.server.HandshakeInterceptor;

/**
 * 该类配置将启用哪些 WebSocketHandler
 */
@Configuration
@EnableWebSocket
public class NekoWebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry webSocketHandlerRegistry) {
        // withSockJS() 表示启用 SockJS 支持
        webSocketHandlerRegistry.addHandler(nekoWebSocketHandler(), "/web/websocket")
                .addInterceptors(nekoHandshakeInterceptor()).withSockJS();
    }

    @Bean
    public WebSocketHandler nekoWebSocketHandler() {
        return new NekoWebSocketHandler();
    }

    @Bean
    public HandshakeInterceptor nekoHandshakeInterceptor() {
        return new NekoHandshakeInterceptor();
    }
}
```
	
由于没有使用 Spring 的 xml 配置文件（同时也没有使用 web.xml），所以要写配置类。如果 Spring 是通过 xml 文件来配置的话，可以参考下面的写法：  

```xml
<websocket:handlers allowed-origins="*">
    <websocket:mapping path="/web/websocket" handler="nekoWebSocketHandler"/>
    <websocket:handshake-interceptors>
        <bean class="com.nekolr.NekoHandshakeInterceptor"/>
    </websocket:handshake-interceptors>
    <websocket:sockjs />
</websocket:handlers>

<bean id="nekoWebSocketHandler" class="com.nekolr.NekoWebSocketHandler"/>
```

MvcConfig 与之前的写法一样，WebInitializer 需要将 NekoWebSocketConfig 加入并注册。与此同时，页面需要引入 sockjs.js 并将 `new WebSocket` 修改为 `new SockJS`。  

![演示](https://img.nekolr.com/images/2019/07/12/7vn.gif)

## 参考
> [JSR 356, Java API for WebSocket](http://www.oracle.com/technetwork/articles/java/jsr356-1937161.html)

> [websocket-spec](https://github.com/javaee/websocket-spec)

> [WebSocketServlet (Apache Tomcat 7.0.94 API Documentation)](http://tomcat.apache.org/tomcat-7.0-doc/api/org/apache/catalina/websocket/WebSocketServlet.html)

> [spring4 websocket + sockjs 遇到的坑](http://wu1g119.iteye.com/blog/2220452)

> [Spring 4.0 系列 9 - websocket 简单应用](http://wiselyman.iteye.com/blog/2003336)