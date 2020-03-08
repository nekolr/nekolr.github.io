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

RabbitMQ 中常用的交换器类型有 fanout、direct、topic 和 headers 这四种，其中 fanout 类型的交换器会把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中，它不处理任何路由规则，直接将消息发送给所有绑定的队列，是四种交换机中速度最快的。direct 类型的交换器则会把消息路由到 BindingKey 与 RoutingKey 完全匹配的队列中，但是有时像 direct 这种严格匹配的方式在很多时候并不能满足实际的需要，因此 topic 类型的交换器则在 direct 的基础上进行了扩展，它约定 RoutingKey 和 BindingKey 都为一个点号分隔的字符串，比如 `com.rabbitmq.client`，BindingKey 中可以存在两种特殊的字符串 `*` 和 `#`，用于模糊匹配，其中 `*` 只能匹配一个单词，而 `#` 可以匹配零个或多个单词。举个例子：比如 BindingKey 为 `*.rabbitmq.*`，那么 Routingkey 必须为类似 `com.rabbitmq.client` 这种的，一个 `*` 号就代表一个单词，如果 BindingKey 为 `java.#`，那么 RoutingKey 就必须为 java 开头的任意字符串，比如 `java.util.concurrent`。至于 headers 类型的交换器则并不依赖路由键的匹配规则来路由消息，而是根据发送消息中的 headers 属性进行匹配。在绑定队列和交换器的时候指定一组键值对，当消息到达交换器时，RabbitMQ 会将该消息的 headers 与绑定时指定的键值对进行比较，完全匹配才会路由到该队列。由于 headers 类型的交换器性能很差，一般不会使用。

## Binding
生产者将消息发送给交换器后，交换器如何知道要将消息发送到哪个队列中呢？答案就是通过绑定。绑定是指将交换器与队列进行关联，在绑定的时候一般还会指定一个绑定键（Binding Key）。生产者在将消息发送给交换器的时候，一般会指定一个 Routing Key，用于表示这个消息的路由规则。这个 Routing Key 需要与交换器类型以及绑定键配合使用。比如一个 direct 类型的交换器，那么消息会路由到 Routing Key 与 Binding Key 完全匹配的某个队列中。所以在交换器类型和绑定键固定的情况下，生产者发送消息时指定的 Routing Key 将最终决定消息的流向。

## Virtual Host
每个 RabbitMQ 服务都可以创建多个 Virtual Host，即虚拟主机。每一个 Virtual Host 本质上都是一个独立的小型 RabbitMQ 服务，拥有自己独立的队列、交换器及绑定关系，并且拥有自己独立的权限。Virtual Host 与 RabbitMQ 服务的关系类似于虚拟机与物理主机的关系，它的存在为各个实例提供了逻辑上的隔离，既能区分 RabbitMQ 中众多的用户，又可以避免队列和交换器等的命名冲突。

# AMQP 简单介绍
AMQP 协议本身包含三层。位于协议最高层的是 Module Layer，主要定义了一些供客户端调用的命令，客户端可以通过这些命令实现自己的业务逻辑。比如，客户端可以通过 Queue.Declare 命令声明一个队列。位于中间层的是 Session Layer，主要负责将客户端的命令发送给服务端，再将服务端的响应发送给客户端，为客户端与服务端之间的通信提供可靠的同步机制和错误处理。位于最底层的是 Transport Layer，主要负责传输二进制数据流，提供帧的处理、信道复用等。

AMQP 说到底还是一个通信协议，通信协议都会涉及到报文的交互，从 low-level 来说，AMQP 协议是应用层的协议，其填充了 TCP 协议层的数据部分。从 high-level 来说，AMQP 协议是通过协议命令进行交互的。AMQP 协议可以看作一系列结构化命令的集合，类似于 HTTP 协议中的方法（GET、POST、DELETE 等）。我们可以运行一段代码，并使用 Wireshark 分析数据包。

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

![AMQP 协议封包](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202002102344/2020/02/10/71e.png)

# 使用介绍
使用 RabbitMQ 的客户端发送和接收消息时，大体上都需要先通过连接工厂创建一个到 Broker 的连接，然后通过连接开启一个信道，由信道声明交换器和队列，再将交换器与队列绑定，然后就可以处理发送或者接收消息的逻辑了。

使用连接工厂创建连接，由连接开启信道不需要过多说明，有一点需要注意的就是通过 Connection 可以创建多个 Channel 实例，但是 Channel 实例不能在线程间共享，也就是说 Channel 实例是非线程安全的。

## 交换器的声明和删除
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

有声明创建交换器的方法，那么当然会有删除的方法，相应的方法如下，其中 ifUnused 用来设置是否在交换器没有被使用的情况下删除，如果设置为 true 则表示只有该交换器没有被使用的情况下才会删除；否则一定会被删除。

```java
Exchange.DeleteOk exchangeDelete(String exchange, boolean ifUnused) throws IOException;
```

## 队列的声明和删除等
声明队列的方法主要有两个，其中一个不带任何参数，默认会创建一个由 RabbitMQ 命名的（类似 amq.gen-j6mNQFoLARlq18lj0LTjWQ 这种，它们也被成为匿名队列）、排他的、自动删除的、非持久化的队列。另一个方法带有很多参数。

```java
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                                 Map<String, Object> arguments) throws IOException;
```

其中 queue 是队列的名称，durable 是设置是否持久化。exclusive 用于设置是否为排他，如果一个队列被声明为排他队列，那么它仅对首次声明它的连接可见，并在连接断开时自动删除。这里需要注意的有三点：首先排他队列是基于连接可见的，即同一个连接的不同信道可以同时访问同一连接创建的排他队列；首次是指如果一个连接已经声明了一个排他队列，那么其他连接是不允许再声明一个同名的排他队列的；即使排他队列是持久化的，一旦连接关闭或者客户端退出，该队列都会自动删除。因此排他队列适用于一个客户端同时处理发送和接收消息的场景。autoDelete 是设置是否为自动删除，它的意思不是当所有连接到此队列的客户端都断开后，这个队列会自动删除。自动删除的前提是至少有一个消费者连接到这个队列，之后所有与该队列连接的消费者都断开，只有在这个前提下自动删除才会生效。因此生产者客户端创建这个队列，或者没有消费者客户端与这个队列连接时，都不会自动删除该队列。arguments 用于设置一些参数，比如 x-message-ttl、x-max-length、x-expires 等。

> 生产者和消费者都可以使用 queueDeclare 来声明一个队列，但是如果消费者在一个信道上订阅了另一个队列，此后就无法在该信道上声明队列了，必须先取消订阅，将信道置为传输模式后才可以继续声明队列。

队列删除的方法与交换器删除的方法类似，有一点不同是多了一个 ifEmpty 参数设置，为 true 时表示只有队列为空（即队列中没有任何消息堆积）时才会进行删除。

```java
Queue.DeleteOk queueDelete(String queue, boolean ifUnused, boolean ifEmpty) throws IOException;
```

还有一个比较特殊的方法，为 queuePurge，该方法用来清空队列中的内容但是不会删除队列本身。

```java
Queue.PurgeOk queuePurge(String queue) throws IOException;
```

## 队列和交换器绑定
```java
Queue.BindOk queueBind(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
```

其中 queue 是要绑定的队列名称，exchange 是要绑定的交换器名称，routingKey 为绑定队列和交换器的路由键，arguments 则定义了绑定的一些参数。

## 交换器和交换器绑定
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

## 发送消息
使用 Channel 类的 basicPublish 方法来发送消息，该方法也有多个重载方法，其中参数最全的方法如下：

```java
void basicPublish(String exchange, String routingKey, boolean mandatory, boolean immediate, BasicProperties props, byte[] body)
        throws IOException;
```

其中 exchange 为交换器名称，如果设置为空字符串则消息会被转发到默认的交换器中，**默认的交换器为 direct 类型，隐式地绑定到了每个队列，并且路由键等于各自绑定的队列名称。不能显式绑定到默认的交换器或与之解绑。默认交换器不能删除**。routingKey 为路由键。当 mandatory 设置为 true，如果交换器出现无法根据自身的类型和路由键匹配一个队列时，那么 RabbitMQ 就会调用 Basic.Return 命令将消息返回给生产者，生产者可以通过 `channel.addReturnListener` 方法添加一个监听器来获取无法匹配的消息；当 mandatory 设置为 false 时，遇到上述情况消息则会被丢弃。当 immediate 设置为 true，如果交换器在将消息路由到队列时发现队列上不存在任何消费者，那么这条消息不会被存入队列中，当与路由键匹配的所有队列中都没有消费者时，消息会通过 Basic.Return 返回至生产者。概括来说就是，mandatory 参数告诉服务端至少要将消息路由到一个队列中，否则消息将返回给生产者。immediate 参数告诉服务端，如果消息匹配的队列上由消费者，则立刻投递，如果所有匹配的队列上都没有消费者，则立即将消息返回给生产者，而不需要将消息存储至队列。

> 需要注意的是，RabbitMQ 3.0 版本开始去掉了对 immediate 参数的支持，官方解释是该参数会影响镜像队列的性能，增加代码的复杂度，建议采用 TTL（过期时间）和 DLX（死信交换器）的方法来替代。

props 为消息的基本属性集，其中包含 14 个属性，包括 contentType、contentEncoding、headers、deliveryMode、priority、correlationld、replyTo、expiration、messageld、timestamp、type、userld、appld、clusterld。

```java
public static class BasicProperties extends com.rabbitmq.client.impl.AMQBasicProperties {
    private String contentType; // 类型
    private String contentEncoding; // 编码
    private Map<String,Object> headers; // 消息头
    private Integer deliveryMode; // 投递模式，1 表示不持久化，2 表示持久化
    private Integer priority; // 优先级
    private String correlationId; // 用于将 RPC 响应与请求相关联
    private String replyTo; // 通常用于命名回调队列
    private String expiration; // 消息的过期时间
    private String messageId;
    private Date timestamp;
    private String type;
    private String userId;
    private String appId;
    private String clusterId;
}
```

## 消费消息
RabbitMQ 的消费模式有两种，分别是推模式（Push）和拉模式（Pull），其中推模式使用 Basic.Consume，拉模式使用 Basic.Get。

### 推模式
在推模式中，消息是由服务端主动推送给消费者的，因此消费者可以通过持续订阅（监听）的方式来获取消息，具体的操作一般是通过实现 Consumer 接口或者继承 DefaultConsumer 类的方式。在调用与 Consumer 相关的方法时，不同的订阅使用不同的消费者标签（consumerTag）来区分，在同一个 Channel 中的消费者也要通过唯一的消费者标签来区分。下面展示一段推模式监听消息的代码，除去声明部分的样板代码。

```java
// 创建消费者
Consumer consumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag,
                               Envelope envelope,
                               AMQP.BasicProperties properties,
                               byte[] body)
            throws IOException {
        // 此处编写处理消息的代码

        // 手动确认
        channel.basicAck(envelope.getDeliveryTag(), false);
    }
};
// 指定队列的消费者，取消自动确认，改为手动确认
channel.basicConsume(queueName, false, "consumerTag1", consumer);
```

除了简单重写 handleDelivery 方法外，还可以重写更多的方法来实现更加复杂的需求，具体可以重写的方法有：

```java
// 当消费者调用 Channel.basicConsume 方法时都会调用，即消费者消费之前都会调用
void handleConsumeOk(String consumerTag);
// 使用 Channel.basicCancel 方法显式取消消费时调用
void handleCancelOk(String consumerTag);
// 当消费者由于其他原因（比如队列被删除）而被取消时调用
void handleCancel(String consumerTag) throws IOException;
// 当 Channel 或者 Connection 关闭时调用
void handleShutdownSignal(String consumerTag, ShutdownSignalException sig);
// 在调用 Channel.basicRecover 方法返回 Basic.recover-ok 时调用，调用之前所有收到但尚未确认的消息将会重发。
void handleRecoverOk(String consumerTag);
```

和生产者一样，消费者客户端同样需要考虑线程安全的问题。消费者客户端的这些重写方法都会被分配到与 Channel 不同的线程上运行，这意味着消费者客户端可以安全地调用一些阻塞方法，比如 channel.queueDeclare、channel.basicCancel 等。每个 Channel 都拥有自己独立的线程，一般最常用的做法就是一个 Channel 对应一个消费者。

### 拉模式
在拉模式中，消费者主动从服务端拉取消息。拉取消息对应在客户端 API 上就是 channel.basicGet 方法，使用该方法可以从服务端获取单条消息。

```java
GetResponse response = channel.basicGet(queueName, false);
if (response == null) {
    // No message retrieved
} else {
    System.out.println(new String(response.getBody()));
    // 手动确认
    channel.basicAck(response.getEnvelope().getDeliveryTag(), false);
}
```

Basic.Consume 会将信道设置为接收模式，直到取消队列的订阅为止。在接收模式期间，服务端会根据 Basic.Qos 的限制不断地给消费者推送消息。如果想从队列获取单条消息而不是持续订阅，可以使用 Basic.Get，但是切记不要将 Basic.Get 放在一个循环中来实现类似 Basic.Consume 的效果，这样做会严重影响 RabbitMQ 的性能。

## 消费端的确认与拒绝
为了保证消息可靠地投递到消费者手中，RabbitMQ 提供了消费者确认机制，消费者在订阅队列时，可以指定 autoAck 参数。当设置 autoAck 为 true 时，RabbitMQ 会自动将发送出去的消息置为确认，然后从内存或磁盘中删除，而不管消费者是否真正消费了该条消息；当设置 autoAck 为 false 时，消费者就不必担心处理消息的过程中消费者进程挂掉后造成消息丢失，RabbitMQ 会一直等到持有消息的消费者显式调用 Basic.Ack 命令为止。

在消费者收到消息后，如果想拒绝当前的消息而不是确认，那么可以使用 Basic.Reject 这个命令，对应的客户端方法为：

```java
void basicReject(long deliveryTag, boolean requeue) throws IOException;
```

其中 deliveryTag 可以看作消息的编号，它是一个 64 位的长整型。如果 requeue 为 true，那么 RabbitMQ 会将这条消息重新存入队列，以便发给下一个消费者；如果 requeue 为 false，那么该消息就会变成死信被丢弃或者发送到死信队列。

Basic.Reject 命令一次只能拒绝一条消息，如果想要批量拒绝消息，可以使用 Basic.Nack 命令，对应的客户端方法为：

```java
void basicNack(long deliveryTag, boolean multiple, boolean requeue) throws IOException;
```

其中 deliveryTag 和 requeue 的含义与 basicReject 方法相同。如果 multiple 为 false，那么该方法只会拒绝编号为 deliveryTag 的这一条消息；如果 multiple 为 true，那么该方法会拒绝 deliveryTag 编号之前所有未被消费者确认的消息。

针对 requeue，AMQP 中还有一个 Basic.Recover 命令具备可重入队列的特征，对应的客户端方法为：

```java
Basic.RecoverOk basicRecover(boolean requeue) throws IOException;
```

这个方法用来请求 RabbitMQ 重新发送还未被确认的消息。如果 requeue 为 true，那么未被确认的消息会被重新加入到队列中，这样对于同一条消息来说，可能会被分配给与之前不同的消费者；如果 requeue 为 false，那么同一条消息就会被分配给与之前相同的消费者。

# RabbitMQ 进阶
RabbitMQ 除了具有消息队列常用的功能外，还有很多高级特性，比如备份交换器、死信队列、延迟队列等。

## 持久化
在 RabbitMQ 中，持久化包括交换器的持久化、队列的持久化以及消息的持久化。交换器的持久化可以在声明交换器时设置，如果交换器不设置持久化，那么在 RabbitMQ 服务重启后，相关交换器的元信息就会丢失。队列的持久化同样也在声明时设置，如果不设置队列持久化，那么在 RabbitMQ 服务重启后，相关队列的元信息也会丢失，同时数据（消息）也会丢失。队列的持久化只能保证队列本身的元信息不会丢失，无法保证队列内部存储的消息不会丢失。要确保消息不会丢失，需要在发送消息时将消息的投递模式，即 deliveryMode 属性设置为 2。只有同时设置了队列和消息的持久化，RabbitMQ 服务重启后消息才不会丢失。

我们可以将所有的消息都设置为持久化，但是这样会严重影响 RabbitMQ 的性能，如果对可靠性要求不高可以选择不持久化消息。即使交换器、队列和消息都设置了持久化，也不能保证消息百分百不丢失。消息从生产到消费经历了生产、转储和消费三个阶段，每个阶段都有可能出现消息丢失的情况。消费阶段出现消息丢失一般是因为消费者订阅队列时设置了 autoAck 为 true，如果此时消费者接收到消息还没来得及处理就宕机了，消息就会丢失，解决方法就是将 autoAck 设置为 false 并进行手工确认。在转储阶段，持久化的消息正确存入 RabbitMQ 后，还需要一段时间（很短，但不能忽视）才能存入磁盘。RabbitMQ 并不会为每条消息都进行同步存盘（调用内核的 fsync 方法），可能仅仅保存到操作系统缓存中，如果这段时间内 RabbitMQ 服务宕机，消息还没来得及落盘就会丢失。解决这种问题的方法就是使用 RabbitMQ 的镜像队列机制，虽然这样不能完全保证消息不会丢失，但是可靠性要比普通的单机模式要高。至于生产阶段，可以在生产者中引入事务机制或者使用发送方确认机制来保证消息已经正确发送并存储到 RabbitMQ 中。

## 生产者确认
默认情况下，生产者发送消息是不会有任何响应信息返回的，也就是默认情况下生产者并不知道消息到底有没有正确到达服务器，如果消息在到达服务器之前就已经丢失，那么持久化操作也就无从谈起。针对这个问题，RabbitMQ 提供了两种解决方式，其一就是事务机制，其二就是发送方确认机制。

### 事务机制
RabbitMQ 客户端中与事务相关的方法有三个，channel.txSelect、channel.txCommit 和 channel.txRollback，其中 channel.txSelect 方法用于将当前信道设置为事务模式，channel.txCommit 方法用于提交事务，channel.txRollback 则用于事务回滚。当信道开启了事务模式后，我们就可以发送消息了，如果事务提交成功，则表示消息一定到达了 RabbitMQ 服务中；如果在事务提交之前由于异常崩溃或其他原因抛出异常，此时就可以将其捕获并进行事务回滚。

```java
try {
    channel.txSelect();
    channel.basicPublish(exchange, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
    channel.txCommit();
} catch (Exception e) {
    e.printStackTrace();
    channel.txRollback();
}
```

### 发送方确认
事务确实能够解决发送方和 RabbitMQ 之间消息确认的问题，但是使用事务会严重影响 RabbitMQ 的性能，因此 RabbitMQ 还提供了一种更轻量的改进方案，它就是发送方确认机制。在该机制中，生产者将信道设置为 confirm（确认）模式，一旦信道进入确认模式，所有在该信道上发布的消息都会被指派一个唯一的 ID（即 deliveryTag，从 1 开始）。在消息被投递到所有匹配的队列之后，RabbitMQ 就会发送一个确认（Basic.Ack）给生产者，确认中包含之前指派的唯一 ID。**如果消息和队列是持久化的，那么确认会在消息写入磁盘之后发出。**此外 RabbitMQ 也可以设置 channel.basicAck 方法中的 multiple 参数，表示这个 ID 序号之前的所有消息都已经得到了处理。

事务机制在发送一条消息后会使发送端阻塞，以等待 RabbitMQ 的响应，之后才能继续发送下一条消息。与之相比，发送方确认机制最大的好处在于它是异步的，生产者发送消息后可以在等信道返回确认的同时继续发送下一条消息。下面使用一段代码简单介绍确认机制的使用：

```java
try {
    // 开启信道的确认模式
    channel.confirmSelect();
    channel.basicPublish("exchange", "routingKey", null, "publisher confirm".getBytes());
    // 等待消息确认响应，可能是 Basic.Ack 也可能是 Basic.Nack
    if (!channel.waitForConfirms()) {
        System.out.println("Send message failed.");
        // 此处编写消息发送失败时需要执行的代码
    }
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

类似 channel.waitForConfirms 的方法共有四种，默认的无参方法返回的条件是客户端收到了相应的回复（Basic.Ack 或 Basic.Nack）或被中断，带有参数 timeout 的方法会在超时时间结束后抛出 `java.util.concurrent.TimeoutException` 异常。剩下两个 waitForConfirmsOrDie 方法在收到 Basic.Nack 回复后会抛出 `java.io.IOException` 异常。

在生产者确认模式中，如果每发送一条消息后就调用 channel.waitForConfirms 方法等待服务端确认，这实际上与事务机制一样还是一种串行同步等待的方式，同时它们的存储确认原理相同，尤其对于持久化的消息来说，两者都需要等待消息落盘（调用 fsync 内核方法）后才会返回。在同步等待的方式下，消息确认机制发送一条消息需要通信交互的命令是 2 条：Basic.Publish 和 Basic.Ack；事务机制是 3 条：Basic.Publish、Tx.Commit 和 Tx.Commit-Ok（或者 Tx.Rollback 和 Tx.Rollback-Ok）。而我们知道，消息确认机制的优势在于并不一定需要同步确认，我们可以采用批量确认或者异步确认的方式。下面针对这两种改进方式分别举例。

在批量确认的方式中，客户端程序需要定期或者定量，亦或者两者结合发送消息后调用 channel.waitForConfirms 方法等待服务端的确认返回。相比之前的一条一确认，批量确认极大地提高了确认的效率，但是当出现返回 Basic.Nack 或者超时的情况时，客户端需要将之前批量发送的消息全部重发，这在网络情况不佳消息经常丢失时性能不升反降。

```java
try {
    channel.confirmSelect();
    // 等待确认的消息总数
    int msgCount = 0;
    while (true) {
        channel.basicPublish("exchange", "routingKey", null, "batch confirm".getBytes());
        // 将发送出去的消息存入缓存中，缓存可以是一个 ArrayList 或者 BlockingQueue 之类的
        if (++msgCount >= BATCH COUNT) {
            msgCount = 0;
            try {
                // 等待响应
                if (channel.waitForConfirms()) {
                    // 将缓存中的消息清空
                } else {
                    // 将缓存中的消息重新发送
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                // 将缓存中的消息重新发送
            }
        }
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

异步确认的方式实现较为复杂，我们需要调用 channel.addConfirmListener 方法为消息确认添加 ConfirmListener 回调接口，该接口中有 handleAck 和 handleNack 两个方法，方法中的 deliveryTag 用来标记消息的唯一有序序号，我们需要为每个信道维护一个意为“unconfirm”的消息序号集合，每发送一条消息，集合元素加一，每当调用 ConfirmListener 接口的 handleAck 方法时，集合删掉一个（multiple 为 false）或多个（multiple 为 true）元素。该集合最好采用有序集合 SortedSet 来维护消息序号。

```java
channel.confirmSelect();
channel.addConfirmListener(new ConfirmListener() {
    @Override
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
        if (multiple) {
            // 如果设置的是 true，则表示这个序号之前的所有消息都得到了处理，因此删除序号之前所有的元素
            unConfirmSet.headSet(deliveryTag - 1).clear();
        } else {
            // 删掉对应的消息序号
            unConfirmSet.remove(deliveryTag);
        }
    }

    @Override
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
        if (multiple) {
            unConfirmSet.headSet(deliveryTag - 1).clear();
        } else {
            unConfirmSet.remove(deliveryTag);
        }
        // 这里需要添加处理消息重发的代码
    }
});

// 模拟一直发送消息的场景
while (true) {
    long nextSeqNo = channel.getNextPublishSeqNo();
    channel.basicPublish("exchange", "routingKey", MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
    // 将序号添加到未处理的集合中
    unConfirmSet.add(nextSeqNo);
}
```

## 消息分发
当队列有多个消费者的时候，队列会将消息以轮询的方式分发，每条消息只会发送给订阅列表中的一个消费者。这种方式非常适合扩展，但是也会有它的缺陷。假如有 n 个消费者，那么 RabbitMQ 会把第 m 条消息分发给第 m % n 个消费者，RabbitMQ 不会理会消费者是否消费并已经确认了消息。如果某些消费者任务繁重，来不及处理这么多的消息，而某些消费者却由于某些原因很快就处理完了分配到的消息，这就会导致应用整体的吞吐量下降。

应对这种情况需要用到 `channel.basicQos(int prefetchCount)` 方法，使用该方法可以限制信道上消费者所能保持的最大未确认消息的数量。举个例子，在订阅队列之前，消费者程序调用了 `channel.basicQos(5)`，之后订阅队列进行消费。RabbitMQ 会保存一份消费者的列表，每发送一条消息都会为对应的消费者计数，如果达到了设定的上限 5，那么 RabbitMQ 就不会向这个消费者再发送任何消息，直到消费者确认了某条消息后，RabbitMQ 将对应的消费者的计数减一，之后该消费者才可以继续接收消息，直到再次达到上限。

basicQos 有三个重载方法，带有 prefetchCount 参数的方法设置为 0 则表示没有上限，还有带有 prefetchSize 参数的方法，该参数表示消费者所能接收未确认消息总体大小的上限，单位为 B，设置为 0 也表示没有上限。对于一个信道来说，它可以同时支持多个队列的消费，当设置了 prefetchCount 大于 0 时，这个信道需要协调各个队列以确保发送的消息都没有超过限定值，这样会使 RabbitMQ 的性能降低，尤其是这些队列分布在集群的多个 Broker 节点上时，因此 RabbitMQ 在 AMQP 0-9-1 协议之上又给 basicQos 方法重新定义了 global 这个参数。

global | AMQP 0-9-1 | RabbitMQ
---|---|---
false | 信道上所有的消费者都需要遵从 prefetchCount 的限制 | 信道上新的消费者需要遵从 prefetchCount 的限制
true | 当前 Connection 上所有的消费者都需要遵从 prefetchCount 的限制 | 信道上所有的消费者都需要遵从 prefetchCount 的限制

对于同一个信道上的多个消费者而言，如果设置了 prefetchCount 的值，那么该设置会对每个消费者都生效，比如下面的例子中，每个消费者各自能接收的未确认消息的上限都是 10。

```java
Consumer consumer1 = ...;
Consumer consumer2 = ...;
// 每个消费者都会限制 10 条
channel.basicQos(10);
channel.basicConsume("queue1", false, consumer1);
channel.basicConsume("queue2", false, consumer2);
```

如果在订阅消息之前，即设置了 global 为 true，又设置了 global 为 false，那么这两个限制都会生效，比如下面的例子：

```java
Consumer consumer1 = ...;
Consumer consumer2 = ...;
// 每个消费者都会限制 3 条
channel.basicQos(3, false);
// 该信道上会限制为 5 条
channel.basicQos(5, true);
channel.basicConsume("queue1", false, consumer1);
channel.basicConsume("queue2", false, consumer2);
```

这个例子中，每个消费者最多只能收到 3 个未确认的消息，并且两个消费者能收到的未确认消息的总和上限为 5，也就是说假如 consumer1 接收到了三条未确认的消息，那么 consumer2 至多只能接收到 2 条未确认的消息，这样设置会额外增加 RabbitMQ 的负担，因为 RabbitMQ 需要更多的资源来协调完成这种限制，因此一般使用默认的 global 设置即可。

## 备份交换器
当生产者在发送消息时设置 mandatory 为 false，那么消息会在未被路由的情况下丢失；如果设置 mandatory 参数为 true，那么还需要编写 ReturnListener 的逻辑，生产者的代码就会变得更加复杂。如果既不想丢失消息又不想使生产者的代码复杂化，那么就可以使用备份交换器（Alternate Exchange）。备份交换器可以在声明交换器的时候添加 alternate-exchange 参数来实现，也可以通过策略（Policy）来实现，两者同时使用时前者的优先级更高。

```java
Map<String, Object> arguments = new HashMap<>();
arguments.put("alternate-exchange", "myAlternateExchange");
channel.exchangeDeclare("normalExchange", "direct", true, false, arguments);
channel.exchangeDeclare("myAlternateExchange", "fanout", true, false, null);
channel.queueDeclare("normalQueue", true, false, false, null);
channel.queueBind("normalQueue", "normalExchange", "normalKey");
channel.queueDeclare("noRouteQueue", true, false, false, null);
channel.queueBind("noRouteQueue", "myAlternateExchange", "");
```

上面的例子中，我们声明了两个交换器，两个交换器各自绑定了一个队列，同时 myAlternateExchange 交换器为 normalExchange 交换器的备份交换器。当路由键为 normalKey 时，消息能够正确路由到 normalQueue 队列中；但是路由键为值时，消息因为不能路由到与 normalExchange 交换器绑定的任意队列上，此时消息就会被发送到备份交换器并最终发送到 noRouteQueue 队列中。

> 需要注意的是，备份交换器与普通的交换器没有太大区别，为了方便使用，一般都会设置为 fanout 类型。如果设置为 direct 类型，那么路由键同样需要进行匹配。

## 死信队列
在 RabbitMQ 中，消息可以设置过期时间（TTL）。我们可以通过队列属性来给队列中的所有消息设置过期时间，也可以针对每条消息单独设置过期时间，两种一起使用时则以 TTL 数值较小的一方为准。一旦消息的生存时间超过设置的 TTL 值，该消息就会变成死信（Dead Message）。**一般来说，消息变成死信除了是由于消息过期以外，还可能是由于消息被拒绝（Basic.Reject 或 Basic.Nack）并设置 requeue 为 false，以及队列达到最大长度。**正常情况下消费者将无法收到变成死信的消息，但是为了避免消息变成死信后丢失，我们可以使用死信队列。

死信队列也可以叫做死信交换器（Dead-Letter-Exchange，DLX），当消息在一个队列中变成死信后，它能够被重新发送到另一个交换器中，这个交换器就是死信交换器，绑定死信交换器的队列就是死信队列。实际上死信交换器与普通的交换器没有区别，它能在任何队列上被指定，实际上设置死信队列就是修改某个队列的属性，在声明队列时加入 `x-dead-letter-exchange` 参数，这样普通队列就变成了死信队列，普通的交换器也就变成了死信交换器。

```java
channel.exchangeDeclare("dlx_exchange", "direct", false, false, null);
Map<String, Object> arguments = new HashMap<>();
// 设置参数，指定死信交换器
arguments.put("x-dead-letter-exchange", "dlx_exchange");
// 也可以给 DLX 指定路由键，如果没有特殊指定则使用原队列的路由键
arguments.put("x-dead-letter-routing-key", "dlx-routing-key");
channel.queueDeclare("dlx_queue", false, false, false, arguments);
```

## 延迟队列
延迟队列中存储的是对应的延迟消息，所谓延迟消息是指当消息发送以后，并不会立刻被消费者接收，而是等待特定时间后，消费者才能拿到这个消息进行消费。RabbitMQ 本身并不直接支持延迟队列的功能，但是我们可以通过 DLX 和 TTL 模拟出延迟队列的功能，比如下面这个例子：

```java
channel.exchangeDeclare("exchange_dlx", "direct", true, false, null);
channel.exchangeDeclare("exchange_normal", "fanout", true, false, null);
Map<String, Object> arguments = new HashMap<>();
arguments.put("x-message-ttl", 10000);// 消息过期时间，10000 毫秒
arguments.put("x-dead-letter-exchange", "exchange_dlx");// 指定死信交换器
arguments.put("x-dead-letter-routing-key", "routingKey");// 给 DLX 指定路由键
channel.queueDeclare("queue_normal", true, false, false, arguments);
channel.queueBind("queue_normal", "exchange_normal", "");
channel.queueDeclare("queue_dlx", true, false, false, arguments);
channel.queueBind("queue_dlx", "exchange_dlx", "routingKey");
// 发送消息 dlx，路由键为 rk
channel.basicPublish("exchange_normal", "rk", MessageProperties.PERSISTENT_TEXT_PLAIN, "dlx".getBytes());
```

这个例子中声明了两个交换器 exchange_normal 和 exchange_dlx，其中 exchange_dlx 是死信交换器，两个交换器各自绑定了一个队列，分别为 queue_normal 和 queue_dlx。队列 queue_normal 设置了 TTL 属性为 10 秒，当消息投递到 exchange_normal 交换器时，最终会被转发到 queue_normal 队列，10 秒后由于没有消费者消费，这条消息就会过期变成死信然后转发到死信交换器，最终投递到死信队列，此时消费者只要获取死信队列中的消息并消费就可以了。

## 优先级队列
在优先级队列中，优先级高的消息具有先被消费的特权。下面举一个例子来说明如何设置消息的优先级：

```java
Map<String, Object> arguments = new HashMap<>();
// 设置队列的最大优先级为 10
arguments.put("x-max-priority", 10);
channel.queueDeclare("queue_priority", true, false, false, arguments);

// 设置消息的优先级为 5
channel.basicPublish("exchange_priority", "rk_priority", 
        new AMQP.BasicProperties.Builder().priority(5).build(), "message".getBytes());
```

上面的例子中首先设置了队列的最大优先级为 10，然后有设置了一条消息的优先级为 5。默认消息的优先级最低为 0，最高为队列设置的最大优先级，优先级高的消息可以被优先消费，当然这个是有前提的：如果消费者的消费速度大于生产者的速度且 Broker 中没有消息堆积的情况下，消息设置优先级也就没有意义，因为这种情况下相当于 Broker 中至多只有一条消息，对一条消息设置优先级没有意义。

## RPC 实现
在 RabbitMQ 中实现 RPC 是比较简单的，客户端发送请求消息，服务端返回响应的消息，为了接收响应的消息还需要在请求消息中指定一个回调队列，但是光有回调队列可不行，因为对于回调队列而言，在其收到一条响应的消息之后，它并不知道这条响应的消息应该和哪个请求相匹配，因此这里就需要用到 `correlationId` 属性，我们只要为每个请求设置一个唯一的 `correlationId`，在回调队列接收到响应的消息时，就可以根据这个唯一值来匹配相应的请求。下面给出 RabbitMQ 官网的一个 RPC 例子：

```java
public class RPCClient implements AutoCloseable {

    private Connection connection;
    private Channel channel;
    private String requestQueueName = "rpc_queue";

    public RPCClient() throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("localhost");
        connection = connectionFactory.newConnection();
        channel = connection.createChannel();
    }

    public static void main(String[] argv) {
        try (RPCClient fibonacciRpc = new RPCClient()) {
            for (int i = 0; i < 32; i++) {
                String i_str = Integer.toString(i);
                System.out.println(" [x] Requesting fib(" + i_str + ")");
                String response = fibonacciRpc.call(i_str);
                System.out.println(" [.] Got '" + response + "'");
            }
        } catch (IOException | TimeoutException | InterruptedException e) {
            e.printStackTrace();
        }
    }

    public String call(String message) throws IOException, InterruptedException {
        // 关联请求的唯一值
        final String corrId = UUID.randomUUID().toString();
        // 声明一个匿名队列作为回调队列
        String replyQueueName = channel.queueDeclare().getQueue();
        AMQP.BasicProperties props = new AMQP.BasicProperties
                .Builder()
                .correlationId(corrId)
                .replyTo(replyQueueName)
                .build();
        // 向 rpc_queue 队列中发送消息
        channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));
        // 使用阻塞队列存放结果
        final BlockingQueue<String> response = new ArrayBlockingQueue<>(1);
        String cTag = channel.basicConsume(replyQueueName, true, (consumerTag, delivery) -> {
            // 如果唯一值匹配则说明是对应的响应
            if (delivery.getProperties().getCorrelationId().equals(corrId)) {
                // 不阻塞
                response.offer(new String(delivery.getBody(), "UTF-8"));
            }
        }, consumerTag -> {
        });
        // 获取结果
        String result = response.take();
        // 取消该消费者，这样队列中就不存在该消费者了
        channel.basicCancel(cTag);
        return result;
    }

    public void close() throws IOException {
        connection.close();
    }
}
```

```java
public class RPCServer {

    private static final String RPC_QUEUE_NAME = "rpc_queue";

    private static int fib(int n) {
        if (n == 0) return 0;
        if (n == 1) return 1;
        return fib(n - 1) + fib(n - 2);
    }

    public static void main(String[] argv) throws Exception {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("localhost");

        try (Connection connection = connectionFactory.newConnection();
             Channel channel = connection.createChannel()) {
            // 声明 rpc_queue 队列
            channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);
            // 清空队列
            channel.queuePurge(RPC_QUEUE_NAME);
            // 一次只能有一个消费者获取到消息
            channel.basicQos(1);
            System.out.println(" [x] Awaiting RPC requests");
            // 锁
            Object monitor = new Object();
            DeliverCallback deliverCallback = (consumerTag, delivery) -> {
                AMQP.BasicProperties replyProps = new AMQP.BasicProperties
                        .Builder()
                        .correlationId(delivery.getProperties().getCorrelationId())
                        .build();
                String response = "";
                try {
                    // 获取 client 发送过来的消息
                    String message = new String(delivery.getBody(), "UTF-8");
                    int n = Integer.parseInt(message);
                    System.out.println(" [.] fib(" + message + ")");
                    response += fib(n);
                } catch (RuntimeException e) {
                    System.out.println(" [.] " + e.toString());
                } finally {
                    // 将计算结果发送给回调队列
                    channel.basicPublish("", delivery.getProperties().getReplyTo(), replyProps, response.getBytes("UTF-8"));
                    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
                    // RabbitMq consumer worker thread notifies the RPC server owner thread
                    synchronized (monitor) {
                        monitor.notify();
                    }
                }
            };
            //
            channel.basicConsume(RPC_QUEUE_NAME, false, deliverCallback, (consumerTag -> { }));
            // Wait and be prepared to consume the message from RPC client.
            while (true) {
                synchronized (monitor) {
                    try {
                        monitor.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

# 参考
> 《RabbitMQ 实战指南》