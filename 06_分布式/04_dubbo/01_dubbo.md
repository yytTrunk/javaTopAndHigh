# Dubbo

## 一 Dubbo工作原理是什么？

| 编号 | 分层                    | 作用                                                         |
| ---- | ----------------------- | ------------------------------------------------------------ |
| 1    | service层，接口层       | 给服务提供者和消费者来实现的                                 |
| 2    | config层，配置层        | 主要是对Dubbo进行各种配置的                                  |
| 3    | proxy层，服务代理层     | 透明生成客户端的stub和服务端的skeleton，通过接口生成动态代理对象 |
| 4    | registry层，服务注册层  | 负责服务的注册与发现                                         |
| 5    | cluster层，集群层       | 封装多个服务提供者的路由以及负载均衡，并桥接注册中心，将多个实例组合成一个服务 |
| 6    | monitor层，监控层       | 对rpc接口的调用次数和调用时间进行监控                        |
| 7    | protocol层，远程调用层  | 封装rpc调用                                                  |
| 8    | exchange层，信息交换层  | 封装请求响应模式，同步转异步                                 |
| 9    | transport层，网络传输层 | 抽象mina和netty为统一接口                                    |
| 10   | serialize层             | 数据序列化层                                                 |

流程图

<img src="https://gitee.com/codeyyt/my_pic/raw/master/image-blog/javath/06/04/04_01.png" alt="Dubbo流程" style="zoom: 67%;" />

消费者

- 动态代理，服务调用者通过接口生成动态代理对象，然后进行调用
- 负载均衡，Cluster层能够感知到提供接口的服务器列表，通过负载均衡，选择一台设备
- 通信协议，依照指定的通信协议（http、rmi、Dubbo）封装RPC调用
- 信息交换，再封装，将RPC调用请求封装成Request
- 网络传输，使用网络通信框架（Netty）,将封装的请求发送到对应的设备端口上，发送前需要序列化



服务提供者

- 网络传输，Server端监听端口，能够接收到服务消费者的请求，并进行反序列化
- 信息交换，解析出封装的Request
- 通信协议，根据规定通信协议解析出调用内容
- （省去了消费者里的负载均衡层）
- 动态代理，根据请求的接口，找到实现接口的类

如果服务注册中心挂掉了，仍然可以通信，因为消费者会将服务提供则的信息拉取到消费者本地。

启动dubbo时，消费者会从zk拉取注册的生产者的地址接口等数据，缓存在本地。每次调用时，按照本地存储的地址进行调用;

**参考**

https://blog.csdn.net/ityouknow/article/details/100789012



分为两块：

- 服务是如何暴露的





- 服务时如何引用的

  consumer从注册中心拉取服务提供信息，





## 二 Dubbo支持哪些通信协议？

**Dubbo协议（默认）**：采用单一长连接，NIO异步通信，基于hessian作为序列化协议。

长连接，一次建立连接，连续使用。

如果使用短连接，每次请求都需要建立一次请求。

适用于：传输数据量小（每次请求在 100kb 以内），但是并发量很高，及服务消费者机器数远大于服务提供者机器数的情况。

**其它**：rmi/hessian/http/webservice等。

- dubbo://（推荐）
- rmi://
- hessian://
- http://

http    hessian   rmi 采用短连接

## 三 Dubbo哪些序列化协议？

Dubbo实际基于不同的通信协议，支持hessian、java二进制序列化、json、SOAP文本序列化等，但是hessian是其默认的序列化协议。

hessian2序列化(默认推荐)：hessian是一种跨语言的高效二进制序列化方式。但这里实际不是原生的hessian2序列化，而是阿里修改过的hessian lite，它是dubbo RPC默认启用的序列化方式

json序列化：目前有两种实现，一种是采用的阿里的fastjson库，另一种是采用dubbo中自己实现的简单json库，但其实现都不是特别成熟，而且json这种文本序列化性能一般不如上面两种二进制序列化。

java序列化：主要是采用JDK自带的Java序列化实现，性能很不理想。



## 四 Dubbo网络传输通信原理？

基于Netty

服务提供者，创建ServerSocketChannel监听对应端口号。Selector轮询ServerSocketChannel，如果有服务消费者需要创建连接时，会创建一个SocketChannel，服务消费者与SocketChannel一一对应。使用多个线程去处理服务消费者的请求，每个线程会对应一个Selector。Selector去轮询查看多个SocketChannel，当存在服务消费者有请求需要处理时，会启动线程进行处理。

线程处理即对应Dubbo服务提供者处理流程。



## 四 Dubbo支持哪些负载均衡策略？

Dubbo内置了4种负载均衡策略:

1. RandomLoadBalance:随机负载均衡。随机的选择一个。是Dubbo的**默认**负载均衡策略。
2. RoundRobinLoadBalance:轮询负载均衡。轮询选择一个。
3. LeastActiveLoadBalance:最少活跃调用数，相同活跃数的随机。活跃数指调用前后计数差。使慢的 Provider 收到更少请求，因为越慢的 Provider 的调用前后计数差会越大。
4. ConsistentHashLoadBalance:一致性哈希负载均衡。相同参数的请求总是落在同一台机器上。

可以调整机器权重，

## 五 Dubbo支持哪些集群容错策略？

Dubbo主要内置了如下几种策略：

- Failover(失败自动切换)（默认）

当调用出现失败的时候，根据配置的重试次数，会自动从其他可用地址中重新选择一个可用的地址进行调用，直到调用成功，或者是达到重试的上限位置。Dubbo里默认配置的重试次数是2。

- Failsafe(失败安全)

即使失败了也不会影响整个调用流程。当出现调用失败时，会忽略此错误，并记录一条日志，同时返回一个空结果，在上游看来调用是成功的。应用场景，可以用于写入审计日志等操作。

- Failfast(快速失败)

失败立即报错，让调用方来决定下一步的操作并保证业务的幂等性。

- Failback(失败自动恢复)

Failback策略中，如果调用失败，则此次失败相当于`Failsafe`，将返回一个空结果。而与`Failsafe`不同的是，Failback策略会将这次调用加入内存中的失败列表中，对于这个列表中的失败调用，会在另一个线程中进行异步重试，重试如果再发生失败，则会忽略，即使重试调用成功，原来的调用方也感知不到了。因此它通常适合于，对于实时性要求不高，且不需要返回值的一些异步操作。

- Forking(并行调用)

一种典型的用成本换时间的思路。即第一次调用的时候就同时发起多个调用，只要其中一个调用成功，就认为成功。在资源充足，且对于失败的容忍度较低的场景下，可以采用此策略。

- Broadcast(广播调用)

在某些场景下，可能需要对服务的所有提供者进行操作，此时可以使用广播调用策略。此策略会逐个调用所有提供者，只要任意有一个提供者出错，则认为此次调用出错。通常用于通知所有提供者更新缓存或日志等本地资源信息。



## 六 Dubbo支持哪些服务路由？

服务路由包含一条路由规则，路由规则决定了服务消费者的调用目标，即规定了服务消费者可调用哪些服务提供者。

Dubbo 提供了三种服务路由实现，分别为条件路由 ConditionRouter、脚本路由 ScriptRouter 和标签路由 TagRouter。



## 七 Dubbo的SPI机制是什么？

**是什么**

SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件META-INF/services中，并由服务加载器读取配置文件，加载自定义实现类。这样可以在运行时，动态为接口替换实现类。因此，能够通过SPI机制扩展Dubbo组件。

比如，存在接口A，实现有A1, A2，A3

但是代码理不存在接口A，但是在使用时，会将接口A对应的实现方法配置在META-INF/services中，然后让工程依赖于提供的服务Jar，在提供运行时，使用接口A时，会去扫描依赖的Jar包，寻找实现了的A方法。

**应用场景**

应用于插件扩展，新开发插件，接入到开源框架里，来扩展功能。

比如：Java中定义了JDBC接口，Java中并没有提供实际的实现类。在项目运行时，依据使用到的数据库类型，引入对应jar，如MySQL为mysql-jdbc-connector.jar。

比如：Dubbo中的协议扩展、负载均衡扩展、注册中心扩展等。



## 八 Dubbo如何进行服务治理，服务降级？

**服务治理**

服务治理主要作用是改变运行时服务的行为和选址逻辑，达到限流，权重配置等目的。

即如何管理服务，使服务能够更加高效运行？

可以生成服务调用链，统计服务访问压力及时长

**服务降级**

服务调用可能出现问题

1) 多个服务之间可能由于服务没有启动或者网络不通，调用中会出现远程调用失败;

2) 服务请求过大，需要停止部分服务以保证核心业务的正常运行；

解决

```java
<dubbo:reference id="xxxService" interface="com.x..service.xxxxService" check="true" async="false" retries="3" timeout="2000" mock="return false" />
```

可以通过配置接口重试次数或者超时时间，出现问题后进行服务降级Dubbo提供了mock配置，可以返回固定值或者自定义mock业务处理类进行返回。



## 九 如何设计一个类似Dubbo的RPC框架？

依据Dubbo的工作原理进行设计

1）可以使用Zookeeper来做服务注册中心

2）消费者如何去服务注册中心获取服务提供者信息？可以主动拉取，或者基于Zookeeper注册监听后通知

3）消费者如何发起服务请求？基于动态代理，调用的接口对应一个代理，代理能够找到服务提供者对应的地址

4）具体哪台设备进行处理？考虑负载均衡

5）如何发送请求网络协议？序列化协议？基于netty网络协议，hessian序列化协议等其它

6）服务提供者如何接收请求？接收到如何处理？通过监听，接收到对应请求后，反序列化，然后进行处理

（也可以包含其它Dubbo包含的特性SPI、服务治理等）



## 十 dubbo服务暴露过程

- 首先将服务的实现封装成一个Invoker，Invoker中封装了服务的实现类。
- 将Invoker封装成Exporter，并缓存起来，缓存里使用Invoker的url作为key，存储到 DubboProtocol 的 exporterMap中
- 将 URL 注册到注册中心，使得消费者可以获取服务
- 服务端Server启动，监听端口。（请求来到时，根据请求信息生成key，到缓存查找Exporter，就找到了Invoker，就可以完成调用。）



实际上就是将服务封装成一个对象存入全局Map中，然后启动一个Netty服务器监听消费者的消费。实际上就是将服务封装成一个对象存入全局Map中，然后启动一个Netty服务器监听消费者的消费。



理解为Invoker就是一个执行体，一个Invoker对应一个接口，这个接口的方法就可以通过Invoker来执行，如DemoService的所有方法都可以通过这个Invoker来执行。

总结

服务暴露需要根据不同的协议去暴露，所以需要执行不同协议对象procotol实现类，每个procotol中有一个Map，key为服务的唯一标识，value为Exporter对象；Exporter对象可以调用getInvoker()得到这个服务的Invoker对象，得到了这个Invoker对象就可以执行具体服务的方法了。

https://aobing.blog.csdn.net/article/details/108345229

## 十一 dubbo服务引用过程

默认情况下，Dubbo 使用懒汉式引入服务，可配置。





服务订阅 -> 服务转化成Invoker -> Transporter使用具体的Client启动服务准备连接 -> 使用Cluster并继续初始化Invoker -> 封装成一个代理返回。

开始引用服务，调用ReferenceConfig的get方法引用服务，ReferenceConfig调用RegistryProtocol并使用具体的Registry（比如Zookeeper）来订阅服务，Registry会通知Directory开始引用服务（异步），也就是将要引用的服务转化成一个Invoker。Directory会使用具体的Protocol（如Dubbo）将引用的服务转化成一个Invoker。Invoker中使用Transporter来初始化一个具体的Client（比如Netty）用来准备和服务端提供者进行通信。RegistryProtocol调用Cluster的合并方法来初始化Invoker，然后ReferenceConfig在Invoker生成之后返回一个服务的代理。



1. ReferenceConfig#init()初始化各种参数，然后调用createProxy()方法来创建代理对象
2. 会判断调用的方式,服务的引入又分为了三种，第一种是本地引入、第二种是直接连接引入远程服务、第三种是通过注册中心引入远程服务。
3. 无论是哪种方式，最终都是返回一个`Invoker`对象出来
4. 封装好的 `invoker`再通过动态代理封装得到代理类



https://aobing.blog.csdn.net/article/details/108461885



## 参考

- [Dubbo官方文档](http://dubbo.apache.org/zh-cn/docs/)







