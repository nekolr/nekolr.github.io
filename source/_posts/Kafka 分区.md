---
title: Kafka 分区
date: 2022/4/26 15:06:0
tags: [Kafka, 消息队列]
categories: [消息队列]
---

主要讨论 Kafka 分区与生产者和消费者之间的分配关系。

<!-- more -->

# 分区与生产者
生产者在往主题发送消息时，首先需要确定这条消息最终要发送到哪个分区上，为此 Kafka 提供了多种选择分区的策略。在 Kafka 2.4 以前，默认的策略是：如果指定了分区，则消息投递到指定分区。如果未指定分区，但是指定了 key，那么通过 `hash(key)` 来计算分区。如果分区和 key 都没有指定，则轮询选择分区。

```java
// ref: org/apache/kafka/clients/producer/KafkaProducer.java
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
    Integer partition = record.partition();
    // 如果指定了分区，则直接使用该分区；否则通过分区策略去选择
    return partition != null ?
            partition :
            partitioner.partition(
                    record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
}
```

```java
// ref: org/apache/kafka/clients/producer/internals/DefaultPartitioner.java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    // 通过元数据获取 topic 下的所有分区
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();
    // 如果 key 为空
    if (keyBytes == null) {
        // 获取主题中递增的一个值
        int nextValue = nextValue(topic);
        // 获取所有的可用分区（可用分区是指该分区存在首领副本）
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        if (availablePartitions.size() > 0) {
            // 将递增的值进行取模运算，即轮询算法
            int part = Utils.toPositive(nextValue) % availablePartitions.size();
            return availablePartitions.get(part).partition();
        } else {
            // no partitions are available, give a non-available partition
            return Utils.toPositive(nextValue) % numPartitions;
        }
    } else {
        // hash the keyBytes to choose a partition
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}

// 该方法可以理解为在消息记录中没有指定 key 的情况下，通过生成一个数来代替 key hash
// 即为主题生成一个随机数，之后就在这个随机数的基础上递增
private int nextValue(String topic) {
    // 缓存获取主题的起始随机数
    AtomicInteger counter = topicCounterMap.get(topic);
    if (null == counter) {
        // 创建随机数
        counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
        AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
        if (currentCounter != null) {
            counter = currentCounter;
        }
    }
    // 递增
    return counter.getAndIncrement();
}
```

在 Kafka 2.4 中，默认的分区器中实现了粘性分区（Sticky Partition），这个就不展开了，具体可以看[这篇译文](https://www.cnblogs.com/huxi2b/p/12540092.html)。

# 分区与消费者

<img src="https://cdn.jsdelivr.net/gh/nekolr/image-hosting@202204271740/2022/04/26/lD9.png" alt="分区与消费者" style="width: 80%" />

在 Kafka 中，一个主题（topic）下可以有一个或多个分区（partition），消费者以分组的形式订阅主题，分组内可以有一个或多个消费者。同一时刻，一条消息只能被同一组内的某个消费者消费。这就意味着，在一个主题下，如果分区数大于消费者的个数，那么必定有消费者同时消费 2 个或以上的分区；如果分区数等于消费者的个数，那么正好一个消费者对应一个分区；如果分区数小于消费者的个数，那么必定有消费者处于空闲状态。

当消费者组成员变更时，包括成员加入或离开（比如 shutdown 或 crash），消费者组订阅的主题数变更时（主要发生在基于正则表达式订阅主题，当有新匹配的主题创建时）以及消费者组订阅的主题分区数变更时，Kafka 都将进行一次分区分配的过程，这个过程也叫做再平衡（rebalance）。再平衡过程中，如何分配分区则需要根据消费者的分区分配策略来实现，它可以通过 `partition.assignment.strategy` 属性来配置，Kafka 默认提供了三种策略：range、roundrobin 和 sticky。

## range
range 策略基于每个主题，按照序号排列可用分区，以字典顺序排列消费者，将分区数除以消费者数，得到每个消费者的分区数。如果没有平均划分，那么最初的几个消费者将有一个额外的分区。

假设有两个消费者 c0 和 c1，两个主题 t0 和 t1，每个主题有三个分区，即 t0p0，t0p1，t0p2，t1p0，t1p1，t1p2。那么使用 range 分配策略得到的结果就是：

```
c0 [t0p0，t0p1，t1p0，t1p1]
c1 [t0p2，t1p2]
```

```java
public Map<String, List<TopicPartition>> assign(Map<String, Integer> partitionsPerTopic,
                                                Map<String, Subscription> subscriptions) {
    // 主题与消费者集合的映射
    Map<String, List<String>> consumersPerTopic = consumersPerTopic(subscriptions);
    // key 为消费者 ID，value 为分配给该消费者的 TopicPartition
    Map<String, List<TopicPartition>> assignment = new HashMap<>();
    // 初始化
    for (String memberId : subscriptions.keySet())
        assignment.put(memberId, new ArrayList<>());

    for (Map.Entry<String, List<String>> topicEntry : consumersPerTopic.entrySet()) {
        String topic = topicEntry.getKey();
        List<String> consumersForTopic = topicEntry.getValue();

        // 该 topic 的 partition 个数
        Integer numPartitionsForTopic = partitionsPerTopic.get(topic);
        if (numPartitionsForTopic == null)
            continue;
        // 对消费者按照字典顺序排序
        Collections.sort(consumersForTopic);
        // 计算每个消费者分到的分区数
        int numPartitionsPerConsumer = numPartitionsForTopic / consumersForTopic.size();
        // 取余，计算剩余分区数
        int consumersWithExtraPartition = numPartitionsForTopic % consumersForTopic.size();

        List<TopicPartition> partitions = AbstractPartitionAssignor.partitions(topic, numPartitionsForTopic);
        for (int i = 0, n = consumersForTopic.size(); i < n; i++) {
            // 剩余分区分配给前面的几个消费者
            int start = numPartitionsPerConsumer * i + Math.min(i, consumersWithExtraPartition);
            int length = numPartitionsPerConsumer + (i + 1 > consumersWithExtraPartition ? 0 : 1);
            assignment.get(consumersForTopic.get(i)).addAll(partitions.subList(start, start + length));
        }
    }
    return assignment;
}
```

## round robin
轮询策略基于所有可用消费者和所有可用分区，与 range 策略最大的不同是它不再局限于某个主题。如果所有的消费者的订阅都是相同的，那么就可以均衡分配。

假设有两个消费者 c0 和 c1，两个主题 t0 和 t1，每个主题有三个分区，即 t0p0，t0p1，t0p2，t1p0，t1p1，t1p2。那么最终的分配结果为：

```
c0 [t0p0，t0p2，t1p1]
c1 [t0p1，t1p0，p1p2]
```

事实上，同组也可以订阅不同的主题。如果组中的每个消费者订阅的主题都不相同，分配的过程仍然使用轮询的方式，若消费者没有订阅主题，那么就要跳过该实例，这有可能会导致分配不平衡。也就是说，消费者组是一个逻辑概念，同组意味着同一时刻分区只能被一个消费者实例消费，换句话说，同组意味着一个分区只能分配给组中的一个消费者。

假设有三个消费者 c0、c1、c2 和三个主题 t0、t1、t2，三个主题分别具有 1、2、3 个分区，因此分区为：t0p0、t1p0、t1p1、t2p0、t2p1、t2p2。如果 c0 订阅 t0，c1 订阅 t0、t1，c2 订阅 t0、t1、t2，那么最终分配的结果为：

```
c0 [t0p0]
c1 [t1p0]
c2 [t1p1，t2p0，t2p1，t2p2]
```

```java
public Map<String, List<TopicPartition>> assign(Map<String, Integer> partitionsPerTopic,
                                                Map<String, Subscription> subscriptions) {
    // subscriptions 是组成员与它订阅的主题的映射

    Map<String, List<TopicPartition>> assignment = new HashMap<>();
    for (String memberId : subscriptions.keySet())
        assignment.put(memberId, new ArrayList<>());

    CircularIterator<String> assigner = new CircularIterator<>(Utils.sorted(subscriptions.keySet()));
    for (TopicPartition partition : allPartitionsSorted(partitionsPerTopic, subscriptions)) {
        final String topic = partition.topic();
        // 如果该消费者没有订阅该 topic，则跳过
        while (!subscriptions.get(assigner.peek()).topics().contains(topic))
            assigner.next();
        // 找到订阅该 TopicPartition 的消费者
        assignment.get(assigner.next()).add(partition);
    }
    return assignment;
}

public List<TopicPartition> allPartitionsSorted(Map<String, Integer> partitionsPerTopic,
                                                Map<String, Subscription> subscriptions) {
    SortedSet<String> topics = new TreeSet<>();
    // 所有 topic 排序
    for (Subscription subscription : subscriptions.values())
        topics.addAll(subscription.topics());

    List<TopicPartition> allPartitions = new ArrayList<>();
    for (String topic : topics) {
        Integer numPartitionsForTopic = partitionsPerTopic.get(topic);
        if (numPartitionsForTopic != null)
            allPartitions.addAll(AbstractPartitionAssignor.partitions(topic, numPartitionsForTopic));
    }
    return allPartitions;
}
```

## sticky
前两种分配策略，如果遇到 rebalance 的情况，分区的调整可能会比较大，而粘性分区策略则可以保证在尽量均衡的前提下减少分配结果的变动。