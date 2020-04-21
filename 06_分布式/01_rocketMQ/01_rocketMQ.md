# RocketMQ

## 一 什么是消息队列

消息中间件一个用于**接收消息**、**存储消息**并**转发消息**的中间件。



## 二 为什么要用消息队列

- 异步处理 - A服务做了什么事情，异步发送一个消息给B/C/D等其它服务。**相比于传统的串行、并行方式，提高了系统吞吐量。**

- 应用解耦 - 系统间通过消息通信，不用调用其他系统进行处理，不用关心其他系统调用。

- 流量削锋 - 有些服务(秒杀)，请求量很高，服务处理不过来，那么请求先放到消息队列里面。**通过消息队列长度控制请求量；可以缓解短时间内的高并发请求。**

- 日志处理 - 解决大量日志传输。

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



## 四 RocketMQ特点？

基于Java语言实现

稳点无单点

集群功能完善，经历过实战

生态圈好，社区活跃



## 五 RocketMQ技术架构？

![RocketMQ技术架构](.\img\01_01.png)

RocketMQ架构上主要分为四部分，如上图所示:

- Producer：消息发布的角色，支持分布式集群方式部署。Producer通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递，投递的过程支持快速失败并且低延迟。
- Consumer：消息消费的角色，支持分布式集群方式部署。支持以push推，pull拉两种模式对消息进行消费。同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。
- NameServer：NameServer是一个非常简单的Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。主要包括两个功能：Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；路由信息管理，每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer,Consumer仍然可以动态感知Broker的路由的信息。
- BrokerServer：Broker主要负责消息的存储、投递和查询以及服务高可用保证，为了实现这些功能，Broker包含了以下几个重要子模块。

1. Remoting Module：整个Broker的实体，负责处理来自clients端的请求。
2. Client Manager：负责管理客户端(Producer/Consumer)和维护Consumer的Topic订阅信息
3. Store Service：提供方便简单的API接口处理消息存储到物理硬盘和查询功能。
4. HA Service：高可用服务，提供Master Broker 和 Slave Broker之间的数据同步功能。
5. Index Service：根据特定的Message key对投递到Broker的消息进行索引服务，以提供消息的快速查询。

> 来源 [Apache RocketMQ开发者指南](https://github.com/apache/rocketmq/tree/master/docs/cn)



## 六 RocketMQ集群模式有哪些？

- #### 多Master多Slave模式-异步复制

每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟（毫秒级），这种模式的优缺点如下：

​	优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，同时Master宕机后，消费者仍然可以从Slave消费，而且此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样；

​	缺点：Master宕机，磁盘损坏情况下会丢失少量消息。

- #### 多Master多Slave模式-同步双写

每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，即只有主备都写成功，才向应用返回成功，这种模式的优缺点如下：

​	优点：数据与服务都无单点故障，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高；

​	缺点：性能比异步复制模式略低（大约低10%左右），发送单个消息的RT会略高，且目前版本在主节点宕机后，备机不能自动切换为主机。



## 七 RocketMQ集群工作流程？

结合部署架构图，描述集群工作流程：

- 启动NameServer，NameServer起来后监听端口，等待Broker、Producer、Consumer连上来，相当于一个路由控制中心。
- Broker启动，跟所有的NameServer保持长连接，定时发送心跳包。心跳包中包含当前Broker信息(IP+端口等)以及存储所有Topic信息。注册成功后，NameServer集群中就有Topic跟Broker的映射关系。
- 收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送消息时自动创建Topic。
- Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取当前发送的Topic存在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所在的Broker建立长连接从而向Broker发消息。
- Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取当前订阅Topic存在哪些Broker上，然后直接跟Broker建立连接通道，开始消费消息。

> 参考 [Apache RocketMQ开发者指南](https://github.com/apache/rocketmq/tree/master/docs/cn)



## 八 RocketMQ特性？

### 8.1 普通消息发送流程



### 8.2 定时消息



### 8.3 顺序消息



### 8.4 事务消息





MQ 的常见问题有：

1. 消息的顺序问题
2. 消息的重复问题

消息丢失







参考

[消息队列基础](https://juejin.im/post/5dd3ff85e51d453fe34dfcc5)

[消息队列常见问题剖析](https://zhuanlan.zhihu.com/p/93512474)

[RocketMQ学习](https://blog.csdn.net/dingshuo168/article/details/102970988)

[Apache RocketMQ开发者指南](https://github.com/apache/rocketmq/tree/master/docs/cn)