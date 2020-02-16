---
title: RabbitMQ 入门
date: 2020/2/8 21:17:0
tags: [消息队列]
categories: [消息队列]
---

最早在 Java 体系中使用最多的消息队列是 ActiveMQ，但是由于其社区不够活跃且历史包袱较重，目前使用者已经越来越少，此时另一个合适的选择就是 RabbitMQ。

<!--more-->

# 安装及准备工作
在 Windows 系统中可以使用 Chocolatey 安装，比较方便。在安装完成后，可以使用 `rabbitmqctl start_app` 命令启动 RabbitMQ 服务。同时可以使用 `rabbitmq-plugins list` 命令列出所有插件的启用状态。然后我们使用 `rabbitmq-plugins enable rabbitmq_management` 命令来启动 RabbitMQ 内置的 web 管理服务插件。web 管理服务默认的端口号为 15672，用户名和密码都是 guest，第一次使用时最好新建一个用户并设置管理员权限后删除 guest 用户。

```
C:\Program Files\RabbitMQ Server\rabbitmq_server-3.8.2\sbin>rabbitmq-plugins list
Listing plugins with pattern ".*" ...
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@nekolr
 |/
[  ] rabbitmq_amqp1_0                  3.8.2
[  ] rabbitmq_auth_backend_cache       3.8.2
[  ] rabbitmq_auth_backend_http        3.8.2
[  ] rabbitmq_auth_backend_ldap        3.8.2
[  ] rabbitmq_auth_backend_oauth2      3.8.2
[  ] rabbitmq_auth_mechanism_ssl       3.8.2
[  ] rabbitmq_consistent_hash_exchange 3.8.2
[  ] rabbitmq_event_exchange           3.8.2
[  ] rabbitmq_federation               3.8.2
[  ] rabbitmq_federation_management    3.8.2
[  ] rabbitmq_jms_topic_exchange       3.8.2
[E*] rabbitmq_management               3.8.2
[e*] rabbitmq_management_agent         3.8.2
[  ] rabbitmq_mqtt                     3.8.2
[  ] rabbitmq_peer_discovery_aws       3.8.2
[  ] rabbitmq_peer_discovery_common    3.8.2
[  ] rabbitmq_peer_discovery_consul    3.8.2
[  ] rabbitmq_peer_discovery_etcd      3.8.2
[  ] rabbitmq_peer_discovery_k8s       3.8.2
[  ] rabbitmq_prometheus               3.8.2
[  ] rabbitmq_random_exchange          3.8.2
[  ] rabbitmq_recent_history_exchange  3.8.2
[  ] rabbitmq_sharding                 3.8.2
[  ] rabbitmq_shovel                   3.8.2
[  ] rabbitmq_shovel_management        3.8.2
[  ] rabbitmq_stomp                    3.8.2
[  ] rabbitmq_top                      3.8.2
[  ] rabbitmq_tracing                  3.8.2
[  ] rabbitmq_trust_store              3.8.2
[e*] rabbitmq_web_dispatch             3.8.2
[  ] rabbitmq_web_mqtt                 3.8.2
[  ] rabbitmq_web_mqtt_examples        3.8.2
[  ] rabbitmq_web_stomp                3.8.2
[  ] rabbitmq_web_stomp_examples       3.8.2
```

# 原理及重要概念
![原理图](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202002092141/2020/02/09/NQY.png)

## Queue
在 RabbitMQ 中，消息只能存储于队列中，而在 Kafka 中消息则是存储在 topic 这一逻辑层面，Kafka 中的队列逻辑只是 topic 实际存储文件的位移标识。RabbitMQ 中的生产者将消息最终投递到队列，消费者从队列获取消息并消费。当多个消费者订阅了同一个队列，这时队列中的消息会被均摊（Round-Robin，即轮询）给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。

## Exchange
Exchange 可以叫做交换器。实际上，生产者并不是直接将消息投递到队列中，而是先经过 Exchnage，再由 Exchange 转发到队列。在 RabbitMQ 中有四种不同类型的交换器，不同类型的交换器有着不同的路由策略。

为什么 RabbitMQ 需要 Exchange 呢？因为 AMQP 中有一个思想就是生产者和消费者解耦。我们只需要定义好 Exchange 的路由策略，生产者不需要关心消息会发送给哪个 Queue 或被哪些消费者消费。这种模式下生产者只面向 Exchange，消费者只面向 Queue。Exchange 定义了消息路由到 Queue 的规则，将各个层面的消息传递隔离开来，每一层只需要关心面向的下一层，从而降低了整体的耦合度。

## Binding
生产者将消息发送给交换器后，交换器如何知道要将消息发送到哪个队列中呢？答案就是通过绑定。绑定是指将交换器与队列进行关联，在绑定的时候一般还会指定一个绑定键（Binding Key）。生产者在将消息发送给交换器的时候，一般会指定一个 Routing Key，用于表示这个消息的路由规则。这个 Routing Key 需要与交换器类型以及绑定键配合使用。比如一个 direct 类型的交换器，那么消息会路由到 Routing Key 与 Binding Key 完全匹配的某个队列中。所以在交换器类型和绑定键固定的情况下，生产者发送消息时指定的 Routing Key 将最终决定消息的流向。

## Virtual Host
每个 RabbitMQ 服务都可以创建多个 Virtual Host，即虚拟主机。每一个 Virtual Host 本质上都是一个独立的小型 RabbitMQ 服务，拥有自己独立的队列、交换器及绑定关系，并且拥有自己独立的权限。Virtual Host 与 RabbitMQ 服务的关系类似于虚拟机与物理主机的关系，它的存在为各个实例提供了逻辑上的隔离，既能区分 RabbitMQ 中众多的用户，又可以避免队列和交换器等的命名冲突。

# AMQP 简单介绍
AMQP 协议本身包含三层。位于协议最高层的是 Module Layer，主要定义了一些供客户端调用的命令，客户端可以通过这些命令实现自己的业务逻辑。比如，客户端可以通过 Queue.Declare 命令声明一个队列。位于中间层的是 Session Layer，主要负责将客户端的命令发送给服务端，再将服务端的响应发送给客户端，为客户端与服务端之间的通信提供可靠的同步机制和错误处理。位于最底层的是 Transport Layer，主要负责传输二进制数据流，提供帧的处理、信道复用等。

AMQP 说到底还是一个通信协议，通信协议都会涉及到报文的交互，从 low-level 来说，AMQP 协议是应用层的协议，其填充了 TCP 协议层的数据部分。从 high-level 来说，AMQP 协议是通过协议命令进行交互的。AMQP 协议可以看作一系列结构化命令的集合，类似于 HTTP 协议中的方法（GET、POST、DELETE 等）。我们可以运行一段代码，并使用 Wrieshark 分析数据包。

```java
public class FanoutExchangeProducer {

    private static final String EXCHANGE_NAME = "fanout_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        // 连接工厂
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("127.0.0.1");
        connectionFactory.setPort(5672);
        connectionFactory.setUsername("guest");
        connectionFactory.setPassword("guest");
        connectionFactory.setVirtualHost("/");
        // 创建连接
        Connection connection = connectionFactory.newConnection();
        // 创建 Channel
        Channel channel = connection.createChannel();
        // 声明非持久化、非自动删除的 Fanout Exchange
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout", false, false, null);
        // 发送消息
        String message = "Hello Fanout Exchange " + new Date().getTime();
        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
        // 关闭 Channel
        channel.close();
        // 关闭连接
        connection.close();
    }
}
```

![封包](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202002102344/2020/02/10/71e.png)

# 重点 API 介绍
使用 RabbitMQ 的客户端发送和接收消息时，大体上都需要先通过连接工厂创建一个连接到 Broker 的连接，然后通过连接开启一个信道，由信道声明交换器和队列，再将交换器与队列绑定，然后就可以处理发送或者接收消息的逻辑了。

使用连接工厂创建连接，由连接开启信道不需要过多说明，有一点需要注意的就是通过 Connection 可以创建多个 Channel 实例，但是 Channel 实例不能在线程间共享，也就是说 Channel 实例是非线程安全的。

## exchangeDeclare
声明交换器的 exchangeDeclare 方法有多个重载方法，这些重载方法都是由下面这个方法中缺省的某些参数构成的。

```java
Exchange.DeclareOk exchangeDeclare(String exchange,
                                          String type,
                                          boolean durable,
                                          boolean autoDelete,
                                          boolean internal,
                                          Map<String, Object> arguments) throws IOException;
```

其中 exchange 是交换器的名称，type 是交换器的类型，有 fanout、direct、topic 和 headers 四种。durable 用于设置是否进行持久化，autoDelete 用于设置是否自动删除。自动删除的意思不是在所有与该交换器连接的客户端都断开后，RabbitMQ 会自动删除该交换器，自动删除的前提是至少有一个队列或交换器与该交换器绑定，之后所有与该交换器绑定的队列或交换器都与该交换器进行了解绑，只有在这个前提下自动删除才会生效。internal 用于设置交换器是否为内置的，客户端无法直接发送消息到内置交换器，只能通过交换器路由到内置交换器的方式发送消息。arguments 用于传递一些结构化的参数，比如 alternate-exchange。

上面声明交换器的方法是有返回值的，为 Exchange.DeclareOk，表示客户端在声明了一个交换器后，需要等待服务端的 Exchange.Declare-Ok 返回命令。同时客户端 API 也提供了不需要服务端返回值的方法 exchangeDeclareNoWait，使用该方法声明的交换器不能紧接着就使用。还有一个非常有用的方法为 exchangeDeclarePassive，可以用它来检测相应的交换器是否存在，如果存在则正常返回；如果不存在则会抛出异常，同时 Channel 也会被关闭。

## exchangeDelete
有声明创建交换器的方法，那么当然会有删除的方法，相应的方法如下，其中 ifUnused 用来设置是否在交换器没有被使用的情况下删除，如果设置为 true 则表示只有该交换器没有被使用的情况下才会删除；否则一定会被删除。

```java
Exchange.DeleteOk exchangeDelete(String exchange, boolean ifUnused) throws IOException;
```

## queueDeclare
声明队列的方法主要有两个，其中一个不带任何参数，默认会创建一个由 RabbitMQ 命名的（类似 amq.gen-j6mNQFoLARlq18lj0LTjWQ 这种，它们也被成为匿名队列）、排他的、自动删除的、非持久化的队列。另一个方法带有很多参数。

```java
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                                 Map<String, Object> arguments) throws IOException;
```

其中 queue 是队列的名称，durable 是设置是否持久化。exclusive 用于设置是否为排他，如果一个队列被声明为排他队列，那么它仅对首次声明它的连接可见，并在连接断开时自动删除。这里需要注意的有三点：首先排他队列是基于连接可见的，即同一个连接的不同信道可以同时访问同一连接创建的排他队列；首次是指如果一个连接已经声明了一个排他队列，那么其他连接是不允许再声明一个同名的排他队列的；即使排他队列是持久化的，一旦连接关闭或者客户端退出，该队列都会自动删除。因此排他队列适用于一个客户端同时处理发送和接收消息的场景。autoDelete 是设置是否为自动删除，它的意思不是当所有连接到此队列的客户端都断开后，这个队列会自动删除。自动删除的前提是至少有一个消费者连接到这个队列，之后所有与该队列连接的消费者都断开，只有在这个前提下自动删除才会生效。因此生产者客户端创建这个队列，或者没有消费者客户端与这个队列连接时，都不会自动删除该队列。arguments 用于设置一些参数，比如 x-message-ttl、x-max-length、x-expires 等。

> 生产者和消费者都可以使用 queueDeclare 来声明一个队列，但是如果消费者在一个信道上订阅了另一个队列，此后就无法在该信道上声明队列了，必须先取消订阅，将信道置为传输模式后才可以继续声明队列。

## queueDelete 和 queuePurge
队列删除的方法与交换器删除的方法类似，有一点不同是多了一个 ifEmpty 参数设置，为 true 时表示只有队列为空（即队列中没有任何消息堆积）时才会进行删除。

```java
Queue.DeleteOk queueDelete(String queue, boolean ifUnused, boolean ifEmpty) throws IOException;
```

还有一个比较特殊的方法，为 queuePurge，该方法用来清空队列中的内容但是不会删除队列本身。

```java
Queue.PurgeOk queuePurge(String queue) throws IOException;
```

## queueBind
```java
Queue.BindOk queueBind(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
```

其中 queue 是要绑定的队列名称，exchange 是要绑定的交换器名称，routingKey 为绑定队列和交换器的路由键，arguments 则定义了绑定的一些参数。

## exchangeBind
我们不仅可以将交换器与队列绑定，还可以将交换器与交换器绑定。

```java
Exchange.BindOk exchangeBind(String destination, String source, String routingKey, Map<String, Object> arguments) throws IOException;
```

其中 destination 可以理解为消息流向的目的交换器，source 可以理解为消息的源头交换器，这里可以举一个例子：

```java
channel.exchangeDeclare("source", "direct", false, true, null);
channel.exchangeDeclare("destination", "fanout", false, true, null);
channel.exchangeBind("destination", "source", "exKey");
channel.queueDeclare("queue", false, false, true, null);
channel.queueBind("queue", "destination", "");
channel.basicPublish("source", "exKey", null, "exToExDemo".getBytes());
```

![交换器绑定交换器](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202002141416/2020/02/14/bBg.png)
