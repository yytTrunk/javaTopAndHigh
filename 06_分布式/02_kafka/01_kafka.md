# kafka

## 一 什么是消息队列?

消息中间件一个用于**接收消息**、**存储消息**并**转发消息**的中间件。

## 二 为什么需要使用消息队列

作用

- 应用解耦 - 系统间通过消息通信，不用调用其他系统进行处理，不用关心其他系统调用。如果B如果出现了故障，对系统A根本没影响，系统A也感觉不到，B自己处理自己的问题。
- 异步处理 - A服务需要做什么事情，异步发送一个消息给B/C/D等其它服务进行处理。相比于传统的串行，能有效缩短接口响应时间。
- 流量削峰 - 有些服务（如秒杀、抢购），请求量很高，服务处理不过来，通过先把请求积压在消息队列里面，系统从消息队列里拉取定长消息进行处理，可以缓解短时间内高并发请求。

应用

- 日志采集 - 解决大量日志传输。
- 消息通讯 - 消息队列一般都内置了高效的通信机制，因此也可以用在纯的消息通讯。比如实现点对点消息队列，或者聊天室等。

## 三 有哪些消息队列？有什么区别？

常见的消息中间件：Kafka、Rocketmq、ActiveMq、RabbitMQ

kafka：
	1、开发语言：     Scala开发
	2、性能、吞吐量： 吞吐量所有MQ里最优秀，QPS十万级、性能毫秒级、支持集群部署
	3、功能：         功能单一
	4、缺点：         丢数据， 因为数据先写入磁盘缓冲区，未直接落盘。机器故障会造成数据丢失
	5、应用场景：     适当丢失数据没有关系、吞吐量要求高、不需要太多的高级功能的场景，比如大数据场景。
RocketMQ：
	1、开发语言：     java开发
	2、性能、吞吐量： 吞吐量高，QPS十万级、性能毫秒级、支持集群部署
	3、功能：         支持各种高级功能，比如说延迟消息、事务消息、消息回溯、死信队列、消息积压等等
	4、缺点：         官方文档相对简单可能是RocketMQ目前唯一的缺点
	5、应用场景：     适当丢失数据没有关系、吞吐量要求高、不需要太多的高级功能的场景，比如大数据场景。RabbitMQ：
	1、开发语言：     Erlang开发
	2、性能、吞吐量： 吞吐量比较低，QPS几万级、性能u秒级、主从架构
	3、功能：         功能单一
	4、缺点：         Erlang小众语言开发，吞吐量低，集群扩展麻烦
	5、应用场景：     中小公司对并发和吞吐量要求不高的场景。



## 四 kafka核心概念

### 4.1 Topic和日志

Topic 就是数据主题，是数据记录发布的地方,可以用来区分业务系统。Kafka中的Topics总是多订阅者模式，一个topic可以拥有一个或者多个消费者来订阅它的数据。

对于每一个topic， Kafka集群都会维持一个分区日志

![](.\img\02_01.png)

每个分区都是有序且顺序不可变的记录集，并且不断地追加到结构化的commit log文件。分区中的每一个记录都会分配一个id号来表示顺序，我们称之为offset，offset用来唯一的标识分区中每一条记录。

Kafka集群保留所有发布的记录—无论他们是否已被消费—并通过一个可配置的参数——保留期限来控制。举个例子， 如果保留策略设置为2天，一条记录发布后两天内，可以随时被消费，两天过后这条记录会被抛弃并释放磁盘空间。Kafka的性能和数据大小无关，所以长时间存储数据没有什么问题.

<img src=".\img\02_03.png" style="zoom: 25%;" />

事实上，在每一个消费者中唯一保存的元数据是offset（偏移量）即消费在log中的位置。偏移量由消费者所控制：通常在读取记录后，消费者会以线性的方式增加偏移量，但是实际上，由于这个位置由消费者控制，所以消费者可以采用任何顺序来消费记录。例如，一个消费者可以重置到一个旧的偏移量，从而重新处理过去的数据；也可以跳过最近的记录，从"现在"开始消费。

**日志中的 partition（分区）作用？**

1. 当日志大小超过了单台服务器的限制，允许日志进行扩展。每个单独的分区都必须受限于主机的文件限制，不过一个主题可能有多个分区，因此可以处理无限量的数据。
2. 可以作为并行的单元集

Kafka 只保证分区内的记录是有序的，而不保证主题中不同分区的顺序。每个 partition 分区按照key值排序足以满足大多数应用程序的需求。但如果你需要总记录在所有记录的上面，可使用仅有一个分区的主题来实现，这意味着每个消费者组只有一个消费者进程。

### 4.2 分布式

日志的分区partition （分布）在Kafka集群的服务器上。每台服务器在处理数据和请求时，共享这些分区。每一个分区都会在已配置的服务器上进行备份，确保容错性.

每个分区都有一台 server 作为 “leader”，零台或者多台server作为 follwers 。leader server 处理一切对 partition （分区）的读写请求，而follwers只需被动的同步leader上的数据。当leader宕机了，followers 中的一台服务器会自动成为新的 leader。每台 server 都会成为某些分区的 leader 和某些分区的 follower，因此集群的负载是平衡的。

### 4.3 生产者

生产者可以将数据发布到所选择的topic（主题）中。生产者负责将记录分配到topic的哪一个 partition（分区）中。通过round-robin做简单的负载均衡，也可以根据消息中的某一个关键字来进行区分。通常第二种方式使用的更多。

### 4.4 消费者

消费者使用一个 消费组（如Consumer Group A）名称来进行标识，发布到topic中的每条记录被分配给订阅消费组中的一个消费者实例.消费者实例可以分布在多个进程中或者多个机器上。

如果所有的消费者实例在同一消费组中，消息记录会负载平衡到每一个消费者实例.

如果所有的消费者实例在不同的消费组中，每条消息记录会广播到所有的消费者进程.

![消费者](.\img\02_02.png)

 如图，2个broker组成的kafka集群，四个分区(P0-P3)和两个消费者组。消费组A有两个消费者，消费组B有四个消费者。

在Kafka中实现消费的方式是**将日志中的分区划分到每一个消费者实例上**，以便在任何时间，**每个实例都是分区中唯一的消费者**。维护消费组中的消费关系由Kafka协议动态处理。如果新的实例加入组，他们将从组中其他成员处接管一些 partition 分区；如果一个实例消失，拥有的分区将被分发到剩余的实例。

| 名词          | 解释                                                         |
| ------------- | ------------------------------------------------------------ |
| Producer      | 消息的生成者                                                 |
| Consumer      | 消息的消费者                                                 |
| ConsumerGroup | 消费者组，可以并行消费Topic中的partition的消息               |
| Broker        | 缓存代理，Kafka集群中的一台或多台服务器统称broker.           |
| Topic         | Kafka处理资源的消息源(feeds of messages)的不同分类           |
| Partition     | Topic物理上的分组，一个topic可以分为多个partion,每个partion是一个有序的队列。partion中每条消息都会被分配一个 有序的Id(offset) |
| Message       | 消息，是通信的基本单位，每个producer可以向一个topic（主题）发布一些消息 |
| Producers     | 消息和数据生成者，向Kafka的一个topic发布消息的 过程叫做producers |
| Consumers     | 消息和数据的消费者，订阅topic并处理其发布的消费过程叫做consumers |

## 五 kafka如何保证高可用？

Kfaka为分布式架构

对于每一个topic， Kafka集群都会维持一个分区日志。

![](.\img\02_01.png)

日志的分区partition （分布）在Kafka集群的服务器上。每台服务器在处理数据和请求时，共享这些分区。每一个分区都会在**已配置的服务器上进行备份**，确保容错性。

![](.\img\02_05.png)

每个分区都有一台 server 作为 “leader”，零台或者多台服务器作为 follwers 。leader server 处理一切对 partition （分区）的读写请求，而follwers只需被动的同步leader上的数据。当leader宕机了，followers 中的一台服务器会自动成为新的 leader。每台 server 都会成为某些分区的 leader 和某些分区的 follower，因此集群的负载是平衡的。



## 六 kafka有哪些应用场景

Apache Kafka是 一个分布式流处理平台。

1. 构造实时流数据管道，它可以在系统或应用之间可靠地获取数据。 (相当于消息队列MQ)
2. 构建实时流式应用程序，对这些流数据进行转换或者影响。 (就是流处理，通过kafka stream topic和topic之间内部进行变化)



## 七 kafka设计思想？

### 7.1 kafka拓扑结构？

![](.\img\02_04.jpg)

### 7.2 kafka发布消息？

producer 采用 push 模式将消息发布到 broker，每条消息都被 append 到 patition 中，属于顺序写磁盘（顺序写磁盘效率比随机写内存要高，保障 kafka 吞吐率）。

### 7.3 消息路由

producer 发送消息到 broker 时，会根据分区算法选择将其存储到哪一个 partition。其路由机制为：

```
1. 指定了 patition，则直接使用；
2. 未指定 patition 但指定 key，通过对 key 的 value 进行hash 选出一个 patition
3. patition 和 key 都未指定，使用轮询选出一个 patition。
```

![](.\img\02_04.png)

流程说明：

```
1. producer 先从 zookeeper 的 "/brokers/.../state" 节点找到该 partition 的 leader
2. producer 将消息发送给该 leader
3. leader 将消息写入本地 log
4. followers 从 leader pull 消息，写入本地 log 后 leader 发送 ACK
5. leader 收到所有 ISR 中的 replica 的 ACK 后，增加 HW（high watermark，最后 commit 的 offset）并向producer发送 ACK
```

> 参考 https://www.cnblogs.com/cyfonly/p/5954614.html

## 八 Kafka常见问题

### 8.1 Kafka中的ISR、AR代表什么？

ISR：与leader保持同步的follower集合

AR：分区的所有副本

### 8.2 kafka为什么会出现重复消费？如何保证幂等性？

kafka重复消费的根本原因是数据消费了，但是offset没更新，导致重复再消费。因为kafka中消费者每消费一条数据，offset会进行更新，下次再消费会根据offset来消费。如果消息消费了，出现了意外状况如重启、网络丢包等问题导致offset未更新，下次再消费时就会出现重复消费。

**如何保证幂等性**

1. 消息可以使用唯一id标识，消费每条消息时，将消费过消息进行存储，在消费时，先去查询一遍是否消费过

### 8.3 如何保证消息不丢失？

**可能存在消息丢失情况**

1. 生产者将消息发送至MQ，消息丢失
2. 生产者将消息发送至MQ，消费者还没来得及消费，MQ挂掉。（例如某个partition的leader挂掉后，但是follower还没有完全同步leader的数据，此时会选举follower为leader，就会出现数据丢失）
3. 消费者从MQ中获取到消息，未处理就挂掉了，但是已经提交了offset，MQ认为已经处理完

**如何保证**

针对情况1：可以设置参数保证生产者写入消息时，需要MQ接收成功并写入至所有副本中，才认为成功，能够保证生产者不丢失数据

针对情况2：配置partition至少有2个副本；保证至少有一个follower能够与leader保持同步；生产者写入消息时，需要MQ接收成功并写入至所有副本中，才认为成功；

针对情况3：关闭消费者自动提交offset，当消费者处理完成之后自己手动提交offset

### 8.4 如何保证消息的顺序性？

1. kafka中单个partition中消息有序
2. 在写入消息时，给消息指定一个key，保证相同key的消息能够被写入到同一个partition中
3. 从partition中取出数据，然后可以根据哈希，分发到不同的内存队列中
4. 每个内存队列对应一个线程，一个线程去处理内存队列中消息，从而保证有序

### 8.5 如何解决消息积压

1）提高消费并行读 

2）批量方式消费

3）丢弃非重要消息 

4）优化消息消费的过程 , 新建一个topic

### 8.6 如何设计一个MQ？

可以参照kafka的设计

1. 需要支持扩容
2. 落盘，保证MQ挂了，数据不丢失
3. 保证高可用，leader、follower机制.
4. 保证数据不丢失

## 参考

- [kafka官网](http://kafka.apachecn.org/)
- [深入浅出理解基于 Kafka 和 ZooKeeper 的分布式消息队列](https://gitbook.cn/books/5ae1e77197c22f130e67ec4e/index.html)






































