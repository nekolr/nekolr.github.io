---
title: Spring Cloud 概述
date: 2020/3/10 15:57:0
tags: [Spring Cloud,Spring]
categories: [Spring Cloud]
---
Spring Cloud 其实就是一个全家桶，建立在 Spring Boot 之上，是微服务系统架构的一站式解决方案，覆盖了微服务的各个核心组件，其中很多组件都是 Netflix 家的。Spring Cloud 的版本号是以英国伦敦地铁站的名字命名的，同时按照字典顺序对应版本的时间顺序，最早的 Release 版本为 Angel，最新的 Release 版本为 Hoxton。

<!--more-->

# 服务注册与发现
[Eureka](https://github.com/Netflix/eureka) 是 Netflix 开发的服务注册与发现组件，它是一个基于 REST 的服务，主要用于运行在 AWS 中以云定位为目的中间层服务器的负载均衡和故障转移。Spring Cloud 将它集成到了 Spring Cloud Netflix 项目中，用于 Spring Cloud 服务注册与发现功能的其中一种解决方案（还可以使用 Consul、Etcd、Zookeeper 等作为服务注册中心）。

> 由于 Eureka 2.x 孵化失败，因此官方不再对 Eureka 开发新的功能，项目转为维护状态。

Eureka 主要包含两大组件：Eureka Server 和 Eureka Client。其中 Eureka Server 主要提供服务注册与发现的功能，Eureka Server 之间通过复制的方式完成服务注册表的同步，形成 Eureka 的高可用。Eureka Client 是客户端程序，主要用于简化与 Eureka Server 的交互，同时它还内置了一个使用轮询算法的负载均衡器。在使用了 Eureka Client 的应用启动后，它会周期性地（默认为 30 秒）向 Eureka Server 发送心跳请求以完成续约（Renew），而 Eureka Server 则会定期（默认为 60 秒）执行一次失效服务检测，检查超过一定时间（默认为 90 秒）没有续约的服务并将该服务从服务注册列表中剔除。与此同时，Eureka Client 还会缓存 Eureka Server 中的信息，即使所有的 Eureka Server 节点都宕机，服务消费者仍可以使用缓存找到服务提供者。

Eureka Client 可以分为两种角色，分别是 Application Service（即服务提供者）和 Application Client（即服务消费者）。Application Service 作为服务的提供者，会将自身提供的服务注册到 Eureka Server 中，而 Application Client 作为服务的消费者，会通过 Eureka Server 发现服务并进行消费。需要注意的是，服务提供者和服务消费者并不是绝对的，因为服务提供者在提供服务的同时也可以作为服务消费者进行消费，同理服务消费者也可以对外提供服务。

![Eureka 架构](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003102049/2020/03/10/Jrn.png)

# 服务调用
在 Spring Cloud 中，服务都是以 HTTP 接口的形式暴露的，因此在调用远程服务的时侯就必须使用 HTTP 客户端。常用的 HTTP 客户端除了 JDK 原生提供的 URLConnection，还有一些第三方的库，比如 Apache HttpClient、OkHttp、RestTemplate 等等。我们接下来要说的就是这个 RestTemplate，它是 Spring 提供的用于进行 HTTP 请求的模板类，在 Spring 框架下位于 spring-web 模块中，在 Spring Boot 框架下则位于 spring-boot-starter-web 模块中。下面列举一个使用 RestTemplate 的小例子：

```java
@Autowired
private RestTemplate restTemplate;
// 如果使用了 Eureka，此处就应该是服务提供者的名称
private static String PROVIDER_URL = "http://localhost:8000";

public boolean checkOrder(Order order) {
    String url = PROVIDER_URL + "/order";
    return restTemplate.getForObject(url, Order.class, order).getStatus() == OrderStatus.COMPLETE.getCode();
}
```

可以看到，在使用了 RestTemplate 之后，调用 HTTP 服务就变得更加简单优雅了，但是我们还不能满足，原因是每次调用远程服务的时候，我们都需要提供服务名称和发起请求等这样的样板代码，这未免也太过繁琐了，我们能不能像在单个进程内调用服务那样直接调用呢？答案是可以的，这就引出了 [Feign](https://github.com/OpenFeign/feign) 这个框架。Feign 是一个轻量级的 Java HTTP 请求调用的框架，它可以以声明式（接口注解）的方式调用 HTTP 请求，通过处理注解将请求模板化，当实际调用时传入参数，进而转化成真正的请求。Feign 有自己的声明式规范，同时也支持 JAX-RS 1/2 规范，而 Spring 官方为了使其适配 Spring 自己的注解规范，也就是 Spring Web MVC 的注解规范，又开发了 [spring-cloud-openfeign](https://github.com/spring-cloud/spring-cloud-openfeign)。下面列举一个使用 Feign 的小例子：

```java
@FeignClient(value = "eureka-client-order-service")
public interface OrderService {

    @RequestMapping(value = "/order", method = RequestMethod.GET)
    Order getOrder(Order order);
}
```

```java
@Autowired
private OrderService orderService;

public boolean checkOrder(Order order) {
    return orderService.getOrder(order).getStatus() == OrderStatus.COMPLETE.getCode();
}
```

# 客户端负载均衡
说起负载均衡，不得不说一下集中式的（服务端）负载均衡和进程内的（客户端）负载均衡，这两种都是业界主流的负载均衡方案，区别在于：集中式的负载均衡是在服务消费者和服务提供者之间的独立的负载均衡设备，它可以是硬件（比如 F5），也可以是软件（比如 Nginx），它负责将消费者的请求通过某种负载均衡策略转发给服务提供者；而进程内的负载均衡是将负载均衡集成到了服务消费者中，消费者从注册中心获取到所有可用的服务地址，然后经过某种负载均衡算法选出一个合适的服务地址发起请求。客户端的负载均衡更适合在微服务中进行远程服务调用时作为本地的负载均衡使用，而像 Nginx 这种集中式的负载均衡设备则更适合服务端实现负载均衡，比如一个 Tomcat 集群，使用 Nginx 作为前端的负载均衡和流量入口。

[Ribbon](https://github.com/Netflix/ribbon) 就是一个由 Netflix 开发的客户端负载均衡组件，它会优先选择在同一个 Zone 并且负载较少的 Eureka Server，然后从该注册中心获取服务注册列表并缓存到本地，待客户端进行 HTTP 请求调用时，默认使用轮询的算法从服务注册列表中选取一个服务地址进行请求。如果单独使用 Ribbon 进行服务调用，则需要使用 HTTP 客户端（比如 RestTemplate）来构建请求，比较繁琐，而 Feign 其实内置了 Ribbon，所以一般我们会直接使用 Feign。

# 服务容错
实际上，微服务之间的调用不可能都是简单的一对一调用，更多时候是复杂的一对多，甚至层层调用，形成一条复杂的调用链，这种情况下如果某个服务出现问题，很容易就会造成整个应用出现问题，这也就是我们常说的**服务雪崩**。

![服务雪崩](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003111808/2020/03/11/WvW.png)

如上图所示，Service A 在某一时刻突然面临大量的请求调用，即便此时服务 A 扛住了请求，但是也不能保证 Service B 和 Service C 都能抗住。假设服务 C 扛不住挂掉了，那么服务 B 的请求就会阻塞，很快服务 B 的资源就会被耗尽，最终服务 B 也会变成不可用，同理，最终服务 A 也会变成不可用。在这个过程中，由于某个服务不可用而最终导致了整条链路上的服务都不可用，这就是服务雪崩。为了应对这个问题，就出现了服务熔断、服务降级、服务隔离（舱壁隔离 Bulkhead Isolation）等技术。

## 服务熔断
服务熔断具体来说就是当下游的服务因为某些原因突然变得不可用或者响应缓慢时，上游服务为了保证自身以及整体服务的可用性，不再继续调用目标服务，而是直接返回，从而能够快速释放资源。如果目标服务情况好转，则会恢复调用。业内常用的熔断机制就是断路器模式。

![断路器模式](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003111808/2020/03/11/LDR.png)

上图是 Martin Fowler 提出的断路器状态转换图，在最开始时断路器处于 closed 状态，当错误达到一定的阈值后，便转换为 open 状态，这时会有一个重置的超时时间，一旦到了这个时间，断路器就会变成半开状态并尝试放行一部分的请求，一旦请求成功，断路器就会变成一开始的关闭状态。

目前比较常见的熔断器有：大名鼎鼎的 Netflix 家的 [Hystrix](https://github.com/Netflix/Hystrix)，只不过与 Eureka 一样，项目处于维护阶段。还有阿里开源的 [Sentinel](https://github.com/alibaba/Sentinel) 和 Hystrix 官方推荐的 [Resilience4J](https://github.com/resilience4j/resilience4j)。

在使用 Hystrix 时，可以通过 `@HystrixCommand` 注解来标注某个方法，这样该方法就会被 Hystrix 以 AOP 的方式包裹起来，每当该方法的调用时间超过指定的时间时，断路器就会中断该方法的调用。但是中断调用就会返回错误，一种更好的使用方式就是提供一个后备方法，在断路器起作用后转而去调用后备方法，从而为用户提供更加友好的响应信息。

```java
@GetMapping("/getOrder")
@HystrixCommand(fallbackMethod = "getOrderFallback")
public Order getOrder(Order order) {
    String url = "http://localhost:8000/order";
    return restTemplate.getForObject(url, Order.class, order);
}

public Order getOrderFallback() {
    Order order = new Order();
    // ...
    return order;
}
```

## 服务降级
需要注意的是，服务熔断和服务降级是两个不同的概念。有一种说法是，服务熔断是服务降级的其中一种方式，还有很多其他的服务降级方式，比如开关降级（又称为埋点）、限流降级等。当然这种说法也是合理的，但是我们可以更深入地探讨服务熔断与服务降级之间的异同。前几天，也就是 9 号那天，美股三大股指开盘大跌触发了熔断机制，上一次触发熔断还是在 1997 年。在股市中，当股指波动幅度达到某个点后，就会触发熔断，此时交易所为了控制风险就会暂停股市交易。与之类似的，软件系统中的熔断也是在某些服务出现问题时，为了避免整个系统故障而采取的一种保护措施。而服务降级有一个比较形象的生活小例子，就是出远门行李箱的例子。平常去不是很远的地方，可能一个小行李箱就足够了，但是如果出一趟远门，可能带的东西就比较多了，如果一股脑的把所有的东西都拿出来放在行李箱中，放不下，这个时候可能就需要把东西分分类，然后比来比去，最后忍痛将一些非必需品舍弃掉。服务降级就是这么一回事，在整个应用资源紧缺的时候，将一些不是特别重要的服务先关掉，待困难过去后再重新开启。

可以看到，其实服务熔断和服务降级的目的是一致的，都是从可用性的角度出发，为了防止整个系统出现问题而采取的一些措施，它们作用的粒度一般都是服务级别（也有更细粒度的，比如做到数据持久层，允许查询但不允许增删改等），最终都会造成用户体验上的一些缺失。当然它们还是有区别的，一般熔断是一个框架级的处理，即每个微服务都需要具有熔断机制，同时熔断一般都是基于策略自动触发的；而降级一般需要根据具体业务进行分析划分，降级一般先从最外围的服务开始，并且降级很多时候需要人工进行干预，当然完全靠人工是不可能的，服务开关、配置中心等都是必要的。

## 服务隔离
服务隔离的思想来源于造船业中舱壁隔离的设计，将船体的内部空间划分为若干个舱室，舱室与舱室之间严密分开，这样即使出现意外导致船舱破损进水也不会影响整个船只的安全。

Hystrix 就是使用舱壁模式来实现的服务隔离，通过舱壁模式来隔离各个依赖服务之间的调用，同时限制对各个依赖服务的并发访问，从而使得各个依赖服务之间互不影响。Hystrix 具体提供了两种隔离方式，一种是线程池，另一种就是信号量，这两种也是目前比较主流的服务隔离的实现方式。

线程池的实现方式是将对各个依赖服务的调用交由独立的线程池来处理，Hystrix 会为每个依赖服务创建一个线程池，当某个服务出现问题时，该服务对应的线程池就会被迅速占满，服务调用方也就不会再去调用该服务了。线程池的实现方式虽然可以起到很好的隔离作用，但是也因此增加了计算开销，每个命令的执行都会涉及到排队（queueing）、调度（scheduling）和上下文切换（context switching）。而信号量的实现方式是为各个依赖服务设置信号量（或计数器）来限制并发调用，相当于对各个依赖服务做限流，信号量实现下的任务由当前的用户请求线程直接处理，不涉及线程的上下文切换，也没有了超时控制。

![线程池的实现方式](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003122221/2020/03/12/KPD.png)

![线程池与信号量](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003122221/2020/03/12/rpj.png)

还有一种服务隔离的实现方式，这种实现方式是从用户的角度进行服务的隔离，这就是我们常说的**多租户设计架构**。一个用户我们可以理解为一个系统的使用者，而租户则是将所有的用户按照某种粒度划分到若干个组内，每个组就是该系统的一个租户（tenant）。组的划分粒度需要根据具体的业务场景来分析，比如根据用户是否付费的特征将用户划分为会员和非会员，那么就会产生会员组和非会员组。用户划分完毕后再进行服务隔离就非常简单了，此时的服务隔离也可以理解为用户隔离，划分后的组与组之间形成不同的服务实例，这样即使某个服务挂掉了，也只会影响该分组的用户，而不是所有的用户。

# 服务网关
假如我们开发了一个电商网站，这个网站涉及到了很多个微服务，比如商品服务、订单服务、支付服务等等，此时我们就会面临一个问题，即用户如何去访问这些微服务呢？举个例子，比如说用户在手机上看中了一款商品并进行了下单操作，此时我们的 APP 可能需要调用订单服务、会员服务、库存服务等等。如果我们的业务比较简单，完全可以直接调用各个服务的 API，但是这就会有一个问题，即服务的鉴权、限流、日志监控等等这些逻辑应该怎么做？我们总不能为每个服务都集成一遍这些逻辑吧？因此最理想的做法就是将这些公共的业务抽取出来，放到一个统一的地方去处理。抛开这些不谈，就说如果到了后期，我们的业务越来越复杂，需要的微服务也越来越多，可能我们使用客户端（浏览器或者移动应用）打开一个页面就涉及到上百个微服务的协同工作，如果这个时候还是使用客户端直接调用各个服务 API 的方式，一方面我们的客户端代码会变得很难维护，毕竟这可能会涉及到上百个服务的域名和地址，另一个方面也会造成客户端 HTTP 请求的连接数膨胀增多。除此之外，还有一个问题就是现在很多时候由于各方面的因素，我们的微服务可能是由不同的编程语言编写的，也有可能采用了不同的通信协议，比如 HTTP、Dubbo、Thrift 等，这个时候我们不可能要求客户端去适配这么多的协议，这会导致客户端变得更加复杂。

说了这么多，其实就是想说明为了解决以上种种问题，我们非常需要在客户端与微服务之间引入一个中间服务，这个服务就是我们常说的服务网关（API Gateway）。

![服务网关](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003132235/2020/03/13/gda.png)

目前常见的服务网关除了 Netflix 开发的 [Zuul](https://github.com/Netflix/zuul)，还有 Spring Cloud 官方提供的 [Spring Cloud Gateway](https://github.com/spring-cloud/spring-cloud-gateway)，以及一些基于 Nginx + Lua 二次开发的比如 [OpenResty](https://github.com/openresty/openresty)、[Kong](https://github.com/Kong/kong) 等。Zuul 的内部原理可以简单看作是很多不同功能的 filter 的集合。Zuul 1.x 中使用的是基于 Servlet 的同步 IO，2.x 的版本中引入了 Netty 来实现异步 IO，因此可以实现更高的性能。Spring Cloud Gateway 基于 Spring 5、Spring Boot 2.0 和 Project Reactor，发展的比 Zuul 2.x 要早，比 Zuul 2.x 更早地使用 Netty 实现了异步 IO，可以看作是 Zuul 1.x 的升级替代品。而像 OpenResty、Kong 这样的服务网关则既可以拥有 Nginx 处理 HTTP 的高性能，同时又通过 Lua 这种脚本语言获得了强大的动态扩展和处理能力，但是相对的可维护性较差，将来可能需要维护大量的 Lua 脚本。

> 在很多时候，微服务项目中并不会选择使用 nginx 作为服务网关，因为微服务框架一般都是配套的，这样集成起来就比较容易。使用 Nginx + Lua 的方式虽然性能高、扩展性强，但是需要开发人员了解和维护 Lua 脚本，与直接使用 Spring Cloud 全家桶来说无疑成本更高。

# 配置中心
当我们的微服务逐渐增多，那么多的 Consumer、Provider、Eureka Server、Zuul 等等都有自己的配置，如果这时我们想修改某些服务的配置，我们总不能挨个到这些服务下寻找配置并在修改完成后再重启服务吧？所以此时我们需要一种既能对所有服务的配置文件进行统一管理，又能在服务运行时动态修改服务配置的方法，这种方法就是使用配置中心。目前常见的配置中心有 Spring Cloud Config、Zookeeper、Consul、携程的 Apollo 以及阿里的 Nacos。

Spring Cloud Config 通过 Git 来管理各个服务的配置文件，这样就可以直接利用 Git 来实现配置的权限控制、版本控制和回滚等操作，但是由于 Git 客户端的限制，导致它在读写时性能较差。Spring Cloud Config 总体上可以分为两部分：Config Server 和 Config Client。其中 Config Server 作为配置中心的服务端，负责在客户端请求时从 Git 仓库拉取最新的配置，同时配合 Eureka 和 Spring Cloud Bus 实现服务发现和配置的推送更新。Config Client 则负责在服务需要配置时向服务端发起请求来获取配置，使用 Config Client 的服务几乎不需要修改任何代码，只要求加入一个启动配置文件指明使用 Config Server 上的哪个配置文件即可。

![Spring Cloud Config](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003141603/2020/03/14/Qrv.png)

在进行了统一的配置管理之后，我们还需要考虑一个问题，那就是如何进行配置的更新。我们知道，服务中的配置一般会在服务启动时进行加载，如果我们在服务启动之后修改了 Git 仓库中某些服务的配置文件，我们该如何提醒这些服务配置发生了变动呢？不幸的是 Spring Cloud Config 原生不支持配置的实时推送，但是我们可以很容易地想到利用 Git 的 Webhooks 来实现当配置变动时触发相应的刷新操作。此时 Config Client 需要提供一个获取配置变更通知的接口，这样当 Git 仓库中的配置文件发生改变时，就会调用 Webhook 中配置的该接口，Config Client 在接收到该接口请求时就可以主动向 Config Server 请求获取新的配置了。

但是此时又出现了一个新的问题，即如果我们有多个微服务（也就是有多个 Config Client），那么每个服务都需要提供一个通知刷新配置的接口，这样 Webhooks 就会随着服务的增多而越来越难以维护。此时引入消息中间件就成了一个比较合适的选择，因为消息中间件可以将消息路由到一个或多个目的地。Spring Cloud Bus 就是这样的一个组件。

在下图中，我们有一个 Config Server 和微服务 Service A 的三个实例，这三个实例我们都引入了 Spring Cloud Bus，这样它们就都连接到了消息总线上。由于使用 Spring Cloud Bus 的服务都会对外提供一个 URI 为 `/bus/refresh` 的接口用于触发配置变更的通知。因此我们只需要在 Webhooks 中配置一个接口地址即可，这样当我们修改了配置文件之后就会调用该接口触发通知，然后这个消息事件就会被微服务中的其他两个实例从总线中获取到，然后它们就会主动从 Config Server 中获取最新的配置信息。

![某个服务承担配置刷新的职责](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003141756/2020/03/14/OyQ.png)

上面的例子中，我们是通过向某个服务实例请求 `/bus/refresh` 接口的方式来触发总线上其他服务实例的配置刷新，但是在某些特殊场景下（比如灰度发布），我们希望只刷新微服务中的某个实例。此时我们只需要在 `/bus/refresh` 中添加 `destination` 参数即可指定具体的服务实例。也许你觉得到这里就大功告成了，但是其实这里还有优化的空间。与配置中心相比，服务实例产生变动的可能性相对较大。如果我们需要对服务实例进行迁移，那么我们可能不得不去修改 Webhooks 中的配置，因此我们要尽可能地让服务集群中各个节点是对等的，此时我们可以在 Config Server 中也引入 Spring Cloud Bus，然后在 Webhooks 中配置 Config Server 的刷新地址。

![配置中心的 Server 端承担配置刷新的职责](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202003141756/2020/03/14/yPR.png)

# 参考
> [谈谈服务雪崩、降级与熔断](https://www.cnblogs.com/rjzheng/p/10340176.html)

> [谈谈怎么做服务隔离](https://www.cnblogs.com/rjzheng/p/10360454.html)