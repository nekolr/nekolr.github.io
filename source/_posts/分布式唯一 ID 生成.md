---
title: 分布式唯一 ID 生成
date: 2020/5/17 20:30:0
tags: [分布式]
categories: [分布式]
---

在很多分布式系统中，常常需要对大量的数据和消息进行唯一标识。在分库分表的应用中，由于数据库从原先的单一节点切分到了多个节点上，因此数据库自身的主键生成机制就不能再使用了，因为我们无法保证某个分片上的数据库生成的主键在全局上是唯一的。这个时候一个分布式（全局唯一）的主键生成系统就很有必要了。

<!-- more -->

一般的分布式主键生成系统需要满足以下要求。首先是全局唯一，这是最基本的要求。其次要求生成的主键为单调递增或者趋势递增。所谓趋势递增是指从整体上看是递增的趋势，但是有时可能出现两个相邻生成的 ID 是递减的。由于很多关系型数据库都采用 B+ 树来存储索引，主键趋势递增或者单调递增可以充分利用 B+ 树的优势，提高主键的读取和写入性能。最后还有安全性的要求，比如 UUID 可能存在 MAC 地址泄露的风险，或者如果 ID 是单调连续的，那么就很容易被恶意利用，如果是订单号就更危险了，竞争对手可以在两天的相同时间下单，然后通过订单号相减就能够大概知道我们一天的订单量。

# UUID
常见的全局唯一主键生成方案中，UUID 是最简单的一个。标准的 UUID 包含 32 个 16 进制数字，以 8-4-4-4-12 的形式分为五段，比如：550e8400-e29b-41d4-a716-446655440000，目前业界共有五种生成 UUID 的方式，详情请见 IETF 发布的 UUID 规范：[A Universally Unique IDentifier (UUID) URN Namespace](https://www.ietf.org/rfc/rfc4122.txt)。UUID 的优点是通过本地生成，没有网络消耗，性能非常高，缺点是长度过长不容易存储，且可能存在信息安全问题（基于 MAC 地址生成 UUID 的算法可能会造成 MAC 地址泄露）。MySQL 官方对于主键的建议是越短越好，UUID 不符合要求，同时在 InnoDB 引擎中，UUID 的无序性可能会引起数据位置的频繁变动，严重影响性能。

# 数据库
使用单一的数据库进行主键生成，所有的服务都通过该数据库获取唯一主键。在这种方案中可以使用数据库自带的自增机制，或者通过数据库维护一个 sequence 表，表结构类似于：

```sql
CREATE TABLE `SEQUENCE` (
    `table_name` varchar(18) NOT NULL,
    `next_id` bigint(20) NOT NULL,
    PRIMARY KEY (`table_name`)
) ENGINE=InnoDB
```

当需要为某个表生成 ID 时，就从 sequence 表中取出对应表的 next_id，并将 next_id 的值增加 1 后更新到数据库中以备下次使用。这个方案实现简单，但是缺点也很明显，因为所有的插入操作都需要访问这张表，因此该表很容易成为性能瓶颈，同时它也存在单点问题。

# 多数据库实例
针对数据库的单点问题，一个优化方案就是提供多台机器部署，每台机器设置不同的初始值，且步长和机器数目相同。

![多数据库实例](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202005182226/2020/05/17/RrE.png)

这种方案解决了单点问题，但是系统的水平扩展比较困难，在步长确定的情况下，很难通过增加机器实现扩容。同时由于每次获取 ID 都需要读取一次数据库，这对数据库来说压力还是很大。

# Leaf-segment
在数据库方案中，每次获取 ID 都需要读写一次数据库，这对数据库来说压力太大。美团开源的 [Leaf](https://github.com/Meituan-Dianping/Leaf) 在数据库方案的基础上进行了改进，每次都获取一个号段（segment）的值，比如 1 到 1000，当这个号段的 ID 全部使用完毕后再去数据库获取新的号段，通过这种方式可以大大减轻数据库的压力。对应的数据库表结构如下：

```
+-------------+--------------+------+-----+-------------------+-----------------------------+
| Field       | Type         | Null | Key | Default           | Extra                       |
+-------------+--------------+------+-----+-------------------+-----------------------------+
| biz_tag     | varchar(128) | NO   | PRI |                   |                             |
| max_id      | bigint(20)   | NO   |     | 1                 |                             |
| step        | int(11)      | NO   |     | NULL              |                             |
| desc        | varchar(256) | YES  |     | NULL              |                             |
| update_time | timestamp    | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+-------------+--------------+------+-----+-------------------+-----------------------------+
```

其中 biz_tag 是业务名，比如用户业务、订单业务等，max_id 表示当前该业务已经分配的 ID 号段的最大值，step 代表每次分配的号段长度。我们只需要将 step 设置得足够大，比如 1000，那么只有当这 1000 个 ID 被全部消费后才会再次读写数据库。

![Leaf-segment](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202005182226/2020/05/18/wPp.png)

这种方案的优点就是，Leaf 服务可以很方便的线性扩展，因为 Leaf 的状态主要通过数据库来维护，所以 Leaf 可以根据需要扩展出多个一起提供服务。这种本地 JVM 缓存的方式即使在数据库宕机时也能够在短时间内对外提供服务，同时由于 max_id 可以根据需要进行初始化，我们就可以很方便地把业务从原有的 ID 的生成方式上迁移过来。当然缺点也很明显，就是生成的 ID 不够随机，可能存在安全隐患。同时在号段使用完毕后采用同步的方式来读写数据库，这时应用会阻塞在更新数据库的 IO 上，根据网络和数据库的状况，阻塞时间的长短可能会出现较大的波动，如果网络发生抖动，或者数据库产生慢查询都会导致整个系统的响应时间拉长。针对这种情况，Leaf-segment 做了相应的优化，即将获取号段的时机提前，不在号段消耗完毕时去获取，而是当号段消费到某个点（比如 10%）时就异步地把下一个号段加载到内存中。

![双 buffer 优化](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202005182226/2020/05/18/nDW.png)

这种方案采用了双 buffer 的方式，即维护两个 segment 缓存区，在当前号段已经消费 10% 时，如果下一个号段还没有更新，就启动一个线程去更新下一个号段。在当前号段全部消费完毕后，如果下一个号段已经准备完毕，那么就将当前的 segment 缓冲区切换为下一个号段的 segment 缓冲区，如此往复。Leaf 官方推荐将 segment 的长度设置为 QPS 的 600 倍，这样即使数据库宕机，Leaf 仍然可以保证 10 到 20 分钟的可用。

# 雪花算法
snowflake 算法是 twitter 开源的分布式 ID 生成算法，它生成的是一个 64 位的二进制正整数，其中高位的第一位是符号位（sign），我们不使用它，它的值始终是 0。接着是 41 位的时间戳（delta seconds），它存储的不是当前的时间戳，而是从生成器开始使用到当前时间的毫秒差值。接着是 10 位机器标识码（worker node id），意味着可以将该服务部署到 1024 台机器上，如果机器是分机房（IDC）部署的，那么这 10 位可以再拆分成机房 ID + 机器 ID。剩下的 12 位是序列号（sequence），这表示一台机器每毫秒能够产生 4096 个 ID。

> 我们一般取的时间戳是从 1970 年 01 月 01 日 00 时 00 分 00 秒到现在的时间差，如果我们能确定我们取的时间都是某个时间点以后的时间，那么可以将时间戳的起始点设置成这个时间点，从而延长服务的时间。

![雪花算法](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202005182226/2020/05/18/mYz.png)

雪花算法的优点是性能较高，理论上每秒能够产生 409.6 万个 ID。由于时间戳在高位，自增序列在低位，从整体上看 ID 是趋势递增的（之所以不是单调递增的是因为节点 ID 这个因素的影响，比如在同一毫秒内，节点 ID 为 2 的机器先于节点 ID 为 1 的机器生成了的 ID，那么很明显前者大于后者）。同时我们可以根据自身业务的需求，在雪花算法的基础上调整 bit 位的划分。比如我们的单机高峰 QPS 为 10W，那么平均每毫秒的并发量为 100 左右，序列号的位置可以只分配 7 位。雪花算法的缺点是强依赖机器的时钟，如果机器的时钟出现回拨（NTP 时间校准，以及其他因素都有可能导致服务器时钟回拨），就很有可能导致重复 ID 的生成。twitter 官方没有对此给出解决方案，而是简单地抛出了异常。如果只是出现了短暂的回拨，比如 5 毫秒，那么可以等待时钟追回，但是如果出现了大步回拨，那么服务就会出现长时间的不可用。

```scala
if (timestamp < lastTimestamp) {
  exceptionCounter.incr(1)
  log.error("clock is moving backwards.  Rejecting requests until %d.", lastTimestamp);
  throw new InvalidSystemClock("Clock moved backwards.  Refusing to generate id for %d milliseconds".format(lastTimestamp - timestamp));
}
```

# Leaf-snowflake
根据最新的代码库，Leaf 对于机器时钟回拨的处理很有限。由于 Leaf 在美团的服务规模较大，在每台机器手动配置 workId 的成本太高，于是他们引入了 Zookeeper，在 Leaf 服务初始化的时候，通过创建持久连续节点的方式为每台机器和端口号分配一个唯一的 workId，同时在机器的临时文件中缓存一份 workId，保证 Leaf 在 ZooKeeper 出现问题时仍能够正常运行。同时每台机器上的 Leaf 服务会每隔 3 秒定时将系统的时间戳更新到 Zookeeper 对应的节点中，在每次获取 ID 时都会检查当前时间与 3 秒前上报的时间，如果出现回拨会抛出异常，但是从代码来看，实际并没有对这个回拨做任何其他处理。

```java
public boolean init() {
    try {
        // 省略部分代码

        // 检查当前时间是否小于之前上报的时间，如果是则直接抛异常
        if (!checkInitTimeStamp(curator, zk_AddressNode)) {
            throw new CheckLastTimeException("init timestamp check error,forever node timestamp gt this node time");
        }
        // 省略部分代码
    } catch (Exception e) {
        LOGGER.error("Start node ERROR {}", e);
        try {
            // Zookeeper 初始化失败时使用机器临时目录中的配置文件的 workId
            Properties properties = new Properties();
            properties.load(new FileInputStream(new File(PROP_PATH.replace("{port}", port + ""))));
            workerID = Integer.valueOf(properties.getProperty("workerID"));
            LOGGER.warn("START FAILED ,use local node file properties workerID-{}", workerID);
        } catch (Exception e1) {
            LOGGER.error("Read file error ", e1);
            return false;
        }
    }
    return true;
}
```

真正处理回拨的地方是在获取 ID 时，如果时间回拨的偏差不超过 5 毫秒，可以在此处等待 2 倍的偏差时间来等待时钟追回，让上游服务最多多等待 10 毫秒是可以接收的。

```java
// 出现时钟回拨
if (timestamp < lastTimestamp) {
    long offset = lastTimestamp - timestamp;
    if (offset <= 5) {
        try {
            // 偏差不超过 5 ms，等待两倍时间
            wait(offset << 1);
            timestamp = timeGen();
            if (timestamp < lastTimestamp) {
                return new Result(-1, Status.EXCEPTION);
            }
        } catch (InterruptedException e) {
            LOGGER.error("wait interrupted");
            return new Result(-2, Status.EXCEPTION);
        }
    } else {
        return new Result(-3, Status.EXCEPTION);
    }
}
```

# 新思路
理论上 snowflake 可以实现单机每秒产生 409.6 万个 ID，实际上大部分的业务都不太可能有如此高的并发，因此可能会有大量的时间戳被浪费掉，可能在某一毫秒内只生成了几个 ID，此时如果发生了时间回拨，这些被浪费的资源是不是可以利用起来，而不是直接在时钟出现回拨时直接抛出异常。

如果在内存中建立一个数组，数组给一个固定的长度，比如 256（最好是 2 的倍数，方便取模），那么这个数组就可以存储所有最近（256 毫秒内）生成过的 ID 值，如果发生时钟回拨，只需要将回拨后的时间戳对数组长度取模，得到当前位置最近一次生成过的 ID 值，将该值的序列号自增后生成的新 ID 作为当前时间的 ID 值。举个例子，假如当前时间是系统运行的第 768 毫秒，生成过 ID，之后时间回拨到了第 512 毫秒，那么第 512 毫秒有没有生成过 ID 已经不得而知了，但是我们知道第 768 毫秒时生成过 ID，那么直接拿着该 ID 的序列号 + 1 即可得到新 ID 值。再举一个例子，假如还是第 768 毫秒，这个时间可能是最新的时间，也可能是回拨后的时间，但是我们不用去管它，只需要查看第 768 毫秒有没有生成过 ID，也就是去 0 号坑位查看，如果该位置没有值，那么直接用该毫秒生成新值；如果该位置有值，值为第 512 毫秒时生成的 ID，那么直接用第 512 毫秒时生成的 ID 的序列号自增即可；如果该位置有值，值为第 1024 毫秒生成的 ID，那么说明当前出现了时钟回拨，不用管，直接用第 1024 毫秒生成的 ID 的序列号自增即可。

![新思路](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202005201255/2020/05/20/6qQ.png)

当然这种方案也并不是完美的，它假定了一个前提就是服务的 QPS 并不是特别高，在这个前提下，当出现了时钟回退时，它能够在时钟回退到时钟追回这段时间内再多提供 2^12 * 256 = 1048576 个 ID 生成。这里列举一个极限的例子，比如在第 768 毫秒生成的 sequence 为 4095，同时在第 769 毫秒生成的 sequence 为 0，这时发生了时钟回拨，回拨到了第 768 毫秒之前，那么在时间重新到达第 768 毫秒时，由于最近一次生成的 ID 的 sequence 进行自增操作后变成了 0，此时使用下一毫秒的时间和 sequence 来生成 ID 就会出现重复。

# 参考
> [Leaf —— 美团点评分布式 ID 生成系统](https://tech.meituan.com/2017/04/21/mt-leaf.html)

> [美团分布式 ID 生成框架 Leaf 源码分析及优化改进](https://juejin.im/post/5eaea4f4f265da7b991c4c31)

> [关于分布式唯一 ID，snowflake 的一些思考及改进(完美解决时钟回拨问题)](https://www.jianshu.com/p/b1124283fc43)