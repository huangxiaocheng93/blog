# Kafka Broker是什么？

kafka集群中的每个节点都是一个Broker。

# Kafka为什么能实现高吞吐量？

kafka采用给topic分区（partition）的机制来实现高吞吐量，举例来说，kafka把一个topic分为10个partition，那么就可以有10个consumer来同时消费这一个topic，如果是100个partition，那么最多就可以有100个consumer同时消费。

kafka在单个partition上保证消息的顺序性。

# Kafka producer是怎么发送消息的？

> [!IMPORTANT]
> producer会维持和所有的broker的tcp连接，消息都会推送给对应的leader broker.

producer在发送消息时，会先计算出消息的partition，然后找到partition对应的leader broker，然后把消息发送给broker，等待broker返回发送结果。

在Kafka 0.9.0版本以后，消息发送改成了异步流程

# **Kafka采用的是推模式还是拉模式？**

卡夫卡采用的是拉模式，客户端可以设置单次拉取的数量，同时要在超时时间内提交offset，超时未确认的话，kafka会重新投递这批消息；

推拉模式的区别：

Push模式：关注的是时效性；

Pull模式：关注的是消费者的自主性；

# Kafka怎么保证数据的可靠性和一致性

## partition的副本机制

partition机制是为了实现高吞吐量，partition副本机制则是为了保证partition数据的可靠性和一致性。

每个partition的副本中，有一个是leader副本，其他都是follower副本，分别都保存在不同的broker上，leader副本负责处理对外的读写请求，follower副本负责同步leader副本的变更，follower副本的工作就是紧紧的跟上leader副本的进度，当leader副本挂掉的时候顶上成为新的leader。

既然是同步，那就一定会有延迟的问题，所有有一个ISR集合的概念，延迟时间在`replica.lag.time.max.ms` 之内的副本组成的集合就叫ISR集合（in sync replica），超出时间的副本会被踢出ISR集合，直到追上进度。

如果leader挂掉，那么就会从ISR副本中选举出一个新的leader，选举是通过kafka controller来控制的。

选举一般都是选lag值最小的那个副本。

## 数据的可靠性

什么是数据的可靠性，就是消息投递到kafka以后不会丢；

### 消息投递

Producer在生产消息时，可以执行`acks`参数值：

- `acks=0`：生产者不会等待任何来自服务器的响应。消息投递出去，不管服务端有没有收到都直接返回，忽略可靠性获取最大的消息发送吞吐量；
- `acks=1（默认值）`：只要集群的Leader节点收到消息，生产者就会收到一个来自服务器的成功响应，此时说明leader节点已经收到消息了，在吞吐量和可靠性中间找了一个平衡；
- `acks=-1` ：只有kafka isr列表中所有的副本同步数据成功，生产者才会收到一个来自服务器的成功响应，能完全保证可靠性，但是吞吐量会更低；

`acks=-1` 时和参数`min.insync.replicas` （ISR 列表中最小同步副本数）有关系，参数默认为1，那就说明只需要leader副本写入成功就能满足条件，如果我们由3个副本，那么这个值可以设置为2，这样就可以保证至少有一个follow副本同步成功了，为什么不能设置为3呢，如果设置为3，此时刚好有一个broker crash，就会导致消息发送不成功。

### **Topic分区副本**

在消息投递到kafka以后，kafka是通过冗余来保证数据可靠性的，写入的消息会保存在一个partition中，partition会保存几个副本来保证数据的可靠性的，所有的副本里面有一个是leader，负责消息的读写，其他副本是follow，负责同步leader的变更，如果leader挂掉，其中一个follow会重新成为新的leader继续提供服务。

## 数据的一致性

![Image](https://github.com/user-attachments/assets/3c12002c-48b6-4be5-b422-53384fb20c83)

假设分区的副本为3，其中副本0是 Leader，副本1和副本2是 follower，并且在 ISR 列表里面。虽然副本0已经写入了 Message3，但是 Consumer 只能读取到 Message1。因为所有的 ISR 都同步了 Message1，只有 High Water Mark 以上的消息才支持 Consumer 读取，而 High Water Mark 取决于 ISR 列表里面偏移量最小的分区，对应于上图的副本2，这个很类似于木桶原理。

这样做的原因是还没有被足够多副本复制的消息被认为是“不安全”的，如果 Leader 发生崩溃，另一个副本成为新 Leader，那么这些消息很可能丢失了。如果我们允许消费者读取这些消息，可能就会破坏一致性。

# **Kafka的rebalance机制**

## 什么情况下会出现rebalance?

- topic分区个数的增加；
- topic数量发生变化；
- 新的consumer加入consumer group或者已有consumer离开consumer group，这种情况应该是最常遇到的；

### consumer加入consumer group

这种情况应该很好理解，就是增加了新的consumer，需要重新分配消费的partition，注意consumer个数不能超过partition个数，超过了就会存在consumer没有partition可以消费。

### consumer正常离开consumer group

例如服务发布。

### consumer非正常离开consumer group

- consumer未能及时发送心跳导致被踢出导致rebalance，心跳超时时间由配置项`session.timeout.ms` 定义，默认值是10s，consumer每隔`heartbeat.interval.ms` 向group coordinator提交心跳；
    
    需要注意的是：`session.timeout.ms` [和`heartbeat.interval.ms`](http://[和heartbeat.interval.ms](http://xn--heartbeat-pw9o.interval.ms/)) 均是客户端参数；
    
- consumer消费超时而Rebalance，`max.poll.interval.ms` 定义了两次`poll()` 之间的最大间隔，默认值是5分钟，`max.poll.records` 定义了每次拉取的最大消息条数，consumer必须保证在`max.poll.interval.ms` 时间范围内处理完`max.poll.records` 条消息并且提交offset；
    
    需要注意的是：`max.poll.interval.ms` 和`max.poll.records` 均是客户端参数；
    

### 分区数量发生变化

当一个topic增加了partition数量时会触发重平衡。

## rebalance会导致的问题

### 消息重复消费

rebalance一定是由于consumer出现了变化，那么之前被这个consumer拉走的消息会重新投递给新分配的consumer，会造成消息的重复消费，**所以一定要做好幂等措施**。

### 可能导致反复消费同一批消息

如果是因为消费者无法在超时时间内完成消息消费导致的rebalance，那么新分配的consumer也极有可能无法在超时时间内完成消费，有可能导致反复消费同一批消息。

### 造成延迟

一是发生rebalance时，consumer group中的所有消费者都必须停下来等待rebalance完成，这中间就会造成消费的延迟；

二是消息会重复消费，又会造成一些延迟；

# 为什么consumer数量不能大于partition数量

kafka设计如此。一个partition最多只能被一个consumer消费，所以当consumer数量大于partition数量时就会出现有消费者时空闲的。

这样设计的原因时为了保证partition中消息消费的顺序。

这里有个很重要的点，当Kafka的某个consumer group出现积压时，不能简单的增加消费者数量来解决，消费者数量最多只能增加到和partition 1:1，如果还是积压，需要先增加partition数量。

# Group Coordinator是什么

Coordinator：协调者

在kafka的每一个broker上都会创建和开启Coordinator组件，它专门为 Consumer Group 服务，负责**Group Rebalance** 以及提供**位移管理**和**组成员管理**等。

Consumer 端应用程序在提交位移时，其实是向 Coordinator 所在的 Broker 提交位移。同样地，当 Consumer 应用启动时，也是向 Coordinator 所在的 Broker 发送各种请求，然后由 Coordinator 负责执行消费者组的注册、成员管理记录等元数据管理操作。

consumer group应该注册到哪个Coordinator是通过group id计算得到的：

**GroupCoordinatorRequest 请求**

- 第 1 步：确定由位移主题的哪个分区来保存该 Group 数据：partitionId=Math.abs(groupId.hashCode() % offsetsTopicPartitionCount)。
- 第 2 步：找出该分区 Leader 副本所在的 Broker，该 Broker 即为对应的 Coordinator。

# Kafka是CP还是AP系统？

实际上Kafka默认更偏向是AP系统，但是可以通过配置让Kafka变成CP系统。