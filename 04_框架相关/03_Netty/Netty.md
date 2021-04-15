

# Netty架构

<img src="https://s0.lgstatic.com/i/image/M00/60/58/Ciqc1F-NO9KAUOtaAAE1S5uRlDE275.png" alt="Drawing 1.png" style="zoom: 67%;" />

Netty 的逻辑处理架构为典型网络分层架构设计，共分为网络通信层、事件调度层、服务编排层，每一层各司其职。图中包含了 Netty 每一层所用到的核心组件。



## 一 Netty架构分为哪三层？

- 网络通信层：

  网络通信层的职责是执行网络 I/O 的操作。它支持多种网络协议和 I/O 模型的连接操作。当网络数据读取到内核缓冲区后，会触发各种网络事件，这些网络事件会分发给事件调度层进行处理。

  网络通信层的核心组件包含BootStrap、ServerBootStrap、Channel三个组件。

  BootStrap & ServerBootStrap

- 事件调度层

  事件调度层的职责是通过 Reactor 线程模型对各类事件进行聚合处理，通过 Selector 主循环线程集成多种事件（ I/O 事件、信号事件、定时事件等），实际的业务处理逻辑是交由服务编排层中相关的 Handler 完成。

  事件调度层的核心组件包括 EventLoopGroup、EventLoop。

- 服务编排层

  的核心组件包括 ChannelPipeline、ChannelHandler、ChannelHandlerContext。

  ChannelPipeline

## 二 什么是BootStrap、ServerBootStrap、Channel，它们之间有什么关系？

Bootstrap 是“引导”的意思，它主要负责整个 Netty 程序的启动、初始化、服务器连接等过程，它相当于一条主线，串联了 Netty 的其他核心组件。

Netty 中的引导器共分为两种类型：一个为用于客户端引导的 Bootstrap，另一个为用于服务端引导的 ServerBootStrap

Bootstrap 和 ServerBootStrap 十分相似，两者非常重要的区别在于 Bootstrap 可用于连接远端服务器，只绑定一个 EventLoopGroup。而 ServerBootStrap 则用于服务端启动绑定本地端口，会绑定两个 EventLoopGroup，这两个 EventLoopGroup 通常称为 Boss 和 Worker。

Channel 的字面意思是“通道”，它是网络通信的载体。Channel提供了基本的 API 用于网络 I/O 操作，如 register、bind、connect、read、write、flush 等，与底层 Socket 交互能力。

Channel 会有多种状态，如连接建立、连接注册、数据读写、连接销毁等。随着状态的变化，Channel 处于不同的生命周期，每一种状态都会绑定相应的事件回调，表格列举了 Channel 最常见的状态所对应的事件回调。

| 事件                      | 说明                                          |
| ------------------------- | --------------------------------------------- |
| channelRegistered         | Channel 创建后被注册到 EventLoop 上           |
| channelUnregistered       | Channel 创建后未注册或者从 EventLoop 取消注册 |
| channelActive             | Channel 处于就绪状态，可以被读写              |
| channelInactive           | Channel 处于非就绪状态                        |
| channelRead               | Channel 可以从远端读取到数据                  |
| channelReadComplete       | Channel 读取数据完成                          |
| userEventTriggered        | 用户事件触发时                                |
| channelWritabilityChanged | Channel 的写状态发生变化                      |



## 三 什么是EventLoopGroup、EventLoop、Channel 关系

<img src="https://s0.lgstatic.com/i/image/M00/60/64/CgqCHl-NPG6APzDfAAbX5ACAFh8001.png" alt="Drawing 4.png" style="zoom: 33%;" />

从上图中，可以总结出 EventLoopGroup、EventLoop、Channel 的几点关系。

1. 一个 EventLoopGroup 往往包含一个或者多个 EventLoop。EventLoop 用于处理 Channel 生命周期内的所有 I/O 事件，如 accept、connect、read、write 等 I/O 事件。
2. EventLoop 同一时间会与一个线程绑定，每个 EventLoop 负责处理多个 Channel。
3. 每新建一个 Channel，EventLoopGroup 会选择一个 EventLoop 与其绑定。该 Channel 在生命周期内都可以对 EventLoop 进行多次绑定和解绑。

EventLoopGroup 是 Netty 的核心处理引擎，那么 EventLoopGroup 和之前所提到的 Reactor 线程模型到底是什么关系呢？其实 EventLoopGroup 是 Netty Reactor 线程模型的具体实现方式，Netty 通过创建不同的 EventLoopGroup 参数配置，就可以支持 Reactor 的三种线程模型：

1. 单线程模型：EventLoopGroup 只包含一个 EventLoop，Boss 和 Worker 使用同一个EventLoopGroup；
2. 多线程模型：EventLoopGroup 包含多个 EventLoop，Boss 和 Worker 使用同一个EventLoopGroup；
3. 主从多线程模型：EventLoopGroup 包含多个 EventLoop，Boss 是主 Reactor，Worker 是从 Reactor，它们分别使用不同的 EventLoopGroup，主 Reactor 负责新的网络连接 Channel 创建，然后把 Channel 注册到从 Reactor。





## 四  什么是ChannelPipeline、ChannelHandler、ChannelHandlerContext

ChannelPipeline 是 Netty 的核心编排组件，负责组装各种 ChannelHandler，实际数据的编解码以及加工处理操作都是由 ChannelHandler 完成的。ChannelPipeline 可以理解为ChannelHandler 的实例列表——内部通过双向链表将不同的 ChannelHandler 链接在一起。当 I/O 读写事件触发时，ChannelPipeline 会依次调用 ChannelHandler 列表对 Channel 的数据进行拦截和处理。
ChannelPipeline 是线程安全的，因为每一个新的 Channel 都会对应绑定一个新的 ChannelPipeline。一个 ChannelPipeline 关联一个 EventLoop，一个 EventLoop 仅会绑定一个线程。



ChannelPipeline 中包含入站 ChannelInboundHandler 和出站 ChannelOutboundHandler 两种处理器，

客户端和服务端都有各自的 ChannelPipeline。以客户端为例，数据从客户端发向服务端，该过程称为出站，反之则称为入站。数据入站会由一系列 InBoundHandler 处理，然后再以相反方向的 OutBoundHandler 处理后完成出站。我们经常使用的编码 Encoder 是出站操作，解码 Decoder 是入站操作。服务端接收到客户端数据后，需要先经过 Decoder 入站处理后，再通过 Encoder 出站通知客户端。所以客户端和服务端一次完整的请求应答过程可以分为三个步骤：客户端出站（请求数据）、服务端入站（解析数据并执行业务逻辑）、服务端出站（响应结果）。



每创建一个 Channel 都会绑定一个新的 ChannelPipeline，ChannelPipeline 中每加入一个 ChannelHandler 都会绑定一个 ChannelHandlerContext。

<img src="https://s0.lgstatic.com/i/image/M00/60/64/CgqCHl-NPK-ADq0pAABb1k5Zwu8681.png" alt="Drawing 8.png" style="zoom: 67%;" />

ChannelHandlerContext 用于保存 ChannelHandler 上下文，通过 ChannelHandlerContext 我们可以知道 ChannelPipeline 和 ChannelHandler 的关联关系。ChannelHandlerContext 可以实现 ChannelHandler 之间的交互，ChannelHandlerContext 包含了 ChannelHandler 生命周期的所有事件，如 connect、bind、read、flush、write、close 等。



## 五 Netty 各个组件的整体交互流程

<img src="https://s0.lgstatic.com/i/image/M00/60/59/Ciqc1F-NPLeAPdjRAADyud16HmQ759.png" alt="Drawing 9.png" style="zoom:67%;" />

- 服务端启动初始化时有 Boss EventLoopGroup 和 Worker EventLoopGroup 两个组件，其中 Boss 负责监听网络连接事件。当有新的网络连接事件到达时，则将 Channel 注册到 Worker EventLoopGroup。


- Worker EventLoopGroup 会被分配一个 EventLoop 负责处理该 Channel 的读写事件。每个 EventLoop 都是单线程的，通过 Selector 进行事件循环。


- 当客户端发起 I/O 读写事件时，服务端 EventLoop 会进行数据的读取，然后通过 Pipeline 触发各种监听器进行数据的加工处理。


- 客户端数据会被传递到 ChannelPipeline 的第一个 ChannelInboundHandler 中，数据处理完成后，将加工完成的数据传递给下一个 ChannelInboundHandler。


- 当数据写回客户端时，会将处理结果在 ChannelPipeline 的 ChannelOutboundHandler 中传播，最后到达客户端。



## 六 Netty线程模型是什么？

Netty 是采用 Reactor线程模型进行开发。

采用了 I/O 多路复用的方案，Reactor 模式作为其中的事件分发器，负责将读写事件分发给对应的读写事件处理者。

可以使用三种Reactor 模式：单线程模式、多线程模式、主从多线程模式。

- 单线程模式

  > Reactor 的单线程模型结构，在 Reactor 单线程模型中，所有 I/O 操作（包括连接建立、数据读写、事件分发等），都是由一个线程完成的。单线程模型逻辑简单，缺陷也十分明显：
  >
  > 1. 一个线程支持处理的连接数非常有限，CPU 很容易打满，性能方面有明显瓶颈；
  > 2. 当多个事件被同时触发时，只要有一个事件没有处理完，其他后面的事件就无法执行，这就会造成消息积压及请求超时；
  > 3. 线程在处理 I/O 事件时，Select 无法同时处理连接建立、事件分发等操作；
  > 4. 如果 I/O 线程一直处于满负荷状态，很可能造成服务端节点不可用。

  Reactor 单线程模型所有 I/O 操作都由一个线程完成，所以只需要启动一个 EventLoopGroup 即可。

  ```java
  EventLoopGroup group = new NioEventLoopGroup(1);
  ServerBootstrap b = new ServerBootstrap();
  b.group(group)
  ```

- 多线程模式

  >  Reactor 多线程模型将业务逻辑交给多个线程进行处理。除此之外，多线程模型其他的操作与单线程模型是类似的，例如读取数据依然保留了串行化的设计。当客户端有数据发送至服务端时，Select 会监听到可读事件，数据读取完毕后提交到业务线程池中并发处理。

  NioEventLoopGroup 可以不需要任何参数，它默认会启动 2 倍 CPU 核数的线程。

  ```java
  EventLoopGroup group = new NioEventLoopGroup();
  ServerBootstrap b = new ServerBootstrap();
  b.group(group)
  ```

- 主从多线程模式

  <img src="https://s0.lgstatic.com/i/image/M00/64/D5/CgqCHl-ZNDiAPgGOAAHx74H-t44265.png" alt="3.png" style="zoom:50%;" />

  > 主从多线程模型由多个 Reactor 线程组成，每个 Reactor 线程都有独立的 Selector 对象。MainReactor 仅负责处理客户端连接的 Accept 事件，连接建立成功后将新创建的连接对象注册至 SubReactor。再由 SubReactor 分配线程池中的 I/O 线程与其连接绑定，它将负责连接生命周期内所有的 I/O 事件。

  大多数场景下，我们采用的都是主从多线程 Reactor 模型。Boss 是主 Reactor，Worker 是从 Reactor。它们分别使用不同的 NioEventLoopGroup，主 Reactor 负责处理 Accept，然后把 Channel 注册到从 Reactor 上，从 Reactor 主要负责 Channel 生命周期内的所有 I/O 事件。

  ```java
  EventLoopGroup bossGroup = new NioEventLoopGroup();
  EventLoopGroup workerGroup = new NioEventLoopGroup();
  ServerBootstrap b = new ServerBootstrap();
  b.group(bossGroup, workerGroup)
  ```

  主从多线程模式甚至可以适当增加 SubReactor 线程的数量，从而利用多核能力提升系统的吞吐量。



## 七 Netty EventLoop 实现原理

EventLoop 这个概念其实并不是 Netty 独有的，它是一种事件等待和处理的程序模型。

下图展示了 EventLoop 通用的运行模式。每当事件发生时，应用程序都会将产生的事件放入事件队列当中，然后 EventLoop 会轮询从队列中取出事件执行或者将事件分发给相应的事件监听者执行。事件执行的方式通常分为立即执行、延后执行、定期执行几种。

<img src="https://s0.lgstatic.com/i/image/M00/64/C9/Ciqc1F-ZNFKAAZr4AANvWWMqnKw586.png" alt="5.png" style="zoom:67%;" />

### Netty 如何实现 EventLoop

在 Netty 中 EventLoop 可以理解为 Reactor 线程模型的事件处理引擎，每个 EventLoop 线程都维护一个 Selector 选择器和任务队列 taskQueue。它主要负责处理 I/O 事件、普通任务和定时任务。

Netty 中推荐使用 NioEventLoop 作为实现类。

NioEventLoop 每次循环的处理流程都包含事件轮询 select、事件处理 processSelectedKeys、任务处理 runAllTasks 几个步骤，是典型的 Reactor 线程模型的运行机制。而且 Netty 提供了一个参数 ioRatio，可以调整 I/O 事件处理和任务处理的时间比例。

### Netty EventLoop 的实现原理，主要用于事件处理与任务处理

#### 事件处理机制

<img src="https://s0.lgstatic.com/i/image/M00/64/C9/Ciqc1F-ZNGGAcJWSAATcxrhDB1U168.png" alt="6.png" style="zoom: 80%;" />

EventLoop 的事件流转图，以便更好地理解 Netty EventLoop 的设计原理。NioEventLoop 的事件处理机制采用的是无锁串行化的设计思路。

BossEventLoopGroup 和 WorkerEventLoopGroup 包含一个或者多个 NioEventLoop。**BossEventLoopGroup 负责监听客户端的 Accept 事件，当事件触发时，将事件注册至 WorkerEventLoopGroup 中的一个 NioEventLoop** 上。每新建一个 Channel， 只选择一个 NioEventLoop 与其绑定。所以说 Channel 生命周期的所有事件处理都是线程独立的，不同的 NioEventLoop 线程之间不会发生任何交集。

NioEventLoop 完成数据读取后，会调用绑定的 ChannelPipeline 进行事件传播，ChannelPipeline 也是线程安全的，数据会被传递到 ChannelPipeline 的第一个 ChannelHandler 中。数据处理完成后，将加工完成的数据再传递给下一个 ChannelHandler，整个过程是串行化执行，不会发生线程上下文切换的问题。

NioEventLoop 无锁串行化的设计不仅使系统吞吐量达到最大化，而且降低了用户开发业务逻辑的难度，不需要花太多精力关心线程安全问题。虽然单线程执行避免了线程切换，但是它的缺陷就是不能执行时间过长的 I/O 操作，一旦某个 I/O 事件发生阻塞，那么后续的所有 I/O 事件都无法执行，甚至造成事件积压。在使用 Netty 进行程序开发时，我们一定要对 ChannelHandler 的实现逻辑有充分的风险意识。

#### 任务处理机制

NioEventLoop 不仅负责处理 I/O 事件，还要兼顾执行任务队列中的任务。任务队列遵循 FIFO 规则，可以保证任务执行的公平性。NioEventLoop 处理的任务类型基本可以分为三类。

- 普通任务：通过 NioEventLoop 的 execute() 方法向任务队列 taskQueue 中添加任务。例如 Netty 在写数据时会封装 WriteAndFlushTask 提交给 taskQueue。taskQueue 的实现类是多生产者单消费者队列 MpscChunkedArrayQueue，在多线程并发添加任务时，可以保证线程安全。


- 定时任务：通过调用 NioEventLoop 的 schedule() 方法向定时任务队列 scheduledTaskQueue 添加一个定时任务，用于周期性执行该任务。例如，心跳消息发送等。定时任务队列 scheduledTaskQueue 采用优先队列 PriorityQueue 实现。


- 尾部队列：tailTasks 相比于普通任务队列优先级较低，在每次执行完 taskQueue 中任务后会去获取尾部队列中任务执行。尾部任务并不常用，主要用于做一些收尾工作，例如统计事件循环的执行时间、监控信息上报等。

```java
protected boolean runAllTasks(long timeoutNanos) {
    // 1. 合并定时任务到普通任务队列
    fetchFromScheduledTaskQueue();
    // 2. 从普通任务队列中取出任务
    Runnable task = pollTask();
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }
    // 3. 计算任务处理的超时时间
    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        // 4. 安全执行任务
        safeExecute(task);
        runTasks ++;
        // 5. 每执行 64 个任务检查一下是否超时
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }
        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }
    // 6. 收尾工作
    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```



## 八 什么是JDK epoll问题？Netty是如何处理的？



Netty 提供了一种检测机制判断线程是否可能陷入空轮询，具体的实现方式如下：

1. 每次执行 Select 操作之前记录当前时间 currentTimeNanos。

2.  time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos，如果事件轮询的持续时间大于等于 timeoutMillis，那么说明是正常的，否则表明阻塞时间并未达到预期，可能触发了空轮询的 Bug。

2. Netty 引入了计数变量 selectCnt。在正常情况下，selectCnt 会重置，否则会对 selectCnt 自增计数。当 selectCnt 达到 SELECTOR_AUTO_REBUILD_THRESHOLD（默认512） 阈值时，会触发重建 Selector 对象。

   

   Netty 采用这种方法巧妙地规避了 JDK Bug。异常的 Selector 中所有的 SelectionKey 会重新注册到新建的 Selector 上，重建完成之后异常的 Selector 就可以废弃了。

```java
long time = System.nanoTime();
if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
    selectCnt = 1;
} else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
    selector = selectRebuildSelector(selectCnt);
    selectCnt = 1;
    break;
}
```



## 九 EventLoop 最佳实践，使用建议

在日常开发中用好 EventLoop 至关重要，这里结合实际工作中的经验给出一些 EventLoop 的最佳实践方案。

1. 网络连接建立过程中三次握手、安全认证的过程会消耗不少时间。这里建议采用 Boss 和 Worker 两个 EventLoopGroup，有助于分担 Reactor 线程的压力。

2. 由于 Reactor 线程模式适合处理耗时短的任务场景，对于耗时较长的 ChannelHandler 可以考虑维护一个业务线程池，将编解码后的数据封装成 Task 进行异步处理，避免 ChannelHandler 阻塞而造成 EventLoop 不可用。

3. 如果业务逻辑执行时间较短，建议直接在 ChannelHandler 中执行。例如编解码操作，这样可以避免过度设计而造成架构的复杂性。

4. 不宜设计过多的 ChannelHandler。对于系统性能和可维护性都会存在问题，在设计业务架构的时候，需要明确业务分层和 Netty 分层之间的界限。不要一味地将业务逻辑都添加到 ChannelHandler 中。



## 十 粘包/拆包问题

### 为什么会出现粘包问题

TCP 传输协议是面向流的，没有数据包界限。客户端向服务端发送数据时，可能将一个完整的报文拆分成多个小报文进行发送，也可能将多个报文合并成一个大的报文进行发送。因此就有了拆包和粘包。

为什么会出现拆包/粘包现象呢？在网络通信的过程中，每次可以发送的数据包大小是受多种因素限制的，如 MTU 传输单元大小、MSS 最大分段大小、滑动窗口等。如果一次传输的网络包数据大小超过传输单元大小，那么我们的数据可能会拆分为多个数据包发送出去。

<img src="https://s0.lgstatic.com/i/image/M00/67/E0/CgqCHl-iZk2ALa_sAAD704YRY80575.png" alt="Drawing 3.png" style="zoom:67%;" />

在客户端和服务端通信的过程中，服务端一次读到的数据大小是不确定的。如上图所示，拆包/粘包可能会出现以下五种情况：

服务端恰巧读到了两个完整的数据包 A 和 B，没有出现拆包/粘包问题；

服务端接收到 A 和 B 粘在一起的数据包，服务端需要解析出 A 和 B；

服务端收到完整的 A 和 B 的一部分数据包 B-1，服务端需要解析出完整的 A，并等待读取完整的 B 数据包；

服务端接收到 A 的一部分数据包 A-1，此时需要等待接收到完整的 A 数据包；

数据包 A 较大，服务端需要多次才可以接收完数据包 A。

由于拆包/粘包问题的存在，数据接收方很难界定数据包的边界在哪里，很难识别出一个完整的数据包。所以需要提供一种机制来识别数据包的界限，这也是解决拆包/粘包的唯一方法：定义应用层的通信协议。

### 如何解决拆包/粘包？

#### 消息长度固定

每个数据报文都需要一个固定的长度。当接收方累计读取到固定长度的报文后，就认为已经获得一个完整的消息。当发送方的数据小于固定长度时，则需要空位补齐。

消息定长法使用非常简单，但是缺点也非常明显，无法很好设定固定长度的值，如果长度太大会造成字节浪费，长度太小又会影响消息传输，所以在一般情况下消息定长法不会被采用。

#### 特定分隔符

既然接收方无法区分消息的边界，那么我们可以在每次发送报文的尾部加上特定分隔符，接收方就可以根据特殊分隔符进行消息拆分。

由于在发送报文时尾部需要添加特定分隔符，所以对于分隔符的选择一定要避免和消息体中字符相同，以免冲突。否则可能出现错误的消息拆分。比较推荐的做法是将消息进行编码，例如 base64 编码，然后可以选择 64 个编码字符之外的字符作为特定分隔符。特定分隔符法在消息协议足够简单的场景下比较高效，例如大名鼎鼎的 Redis 在通信过程中采用的就是换行分隔符。

#### 消息长度 + 消息内容（最常用）

消息长度 + 消息内容是项目开发中最常用的一种协议，如上展示了该协议的基本格式。消息头中存放消息的总长度，例如使用 4 字节的 int 值记录消息的长度，消息体实际的二进制的字节数据。接收方在解析数据时，首先读取消息头的长度字段 Len，然后紧接着读取长度为 Len 的字节数据，该数据即判定为一个完整的数据报文。依然以上述提到的原始字节数据为例，使用该协议进行编码后的结果如下所示：



## 十一 Netty如何自定义通信协议

目前市面上已经有不少通用的协议，例如 HTTP、HTTPS、JSON-RPC、FTP、IMAP、Protobuf 等。

Netty 常用编码器类型：

- MessageToByteEncoder 对象编码成字节流；


- MessageToMessageEncoder 一种消息类型编码成另外一种消息类型。


Netty 常用解码器类型：

- ByteToMessageDecoder/ReplayingDecoder 将字节流解码为消息对象；


- MessageToMessageDecoder 将一种消息类型解码为另外一种消息类型。

#### 抽象解码类

解码器实现的难度要远大于编码器，因为解码器需要考虑拆包/粘包问题。由于接收方有可能没有接收到完整的消息，所以解码框架需要对入站的数据做缓冲操作，直至获取到完整的消息。



## 十二 Netty 支持哪些常用的解码器？

Netty 提供了很多开箱即用的解码器，这些解码器基本覆盖了 TCP 拆包/粘包的通用解决方案。

#### 固定长度解码器 FixedLengthFrameDecoder

固定长度解码器 FixedLengthFrameDecoder 非常简单，直接通过构造函数设置固定长度的大小 frameLength，无论接收方一次获取多大的数据，都会严格按照 frameLength 进行解码。如果累积读取到长度大小为 frameLength 的消息，那么解码器认为已经获取到了一个完整的消息。如果消息长度小于 frameLength，FixedLengthFrameDecoder 解码器会一直等后续数据包的到达，直至获得完整的消息。

#### 特殊分隔符解码器 DelimiterBasedFrameDecoder

#### 长度域解码器 LengthFieldBasedFrameDecoder

长度域解码器 LengthFieldBasedFrameDecoder 是解决 TCP 拆包/粘包问题最常用的**解码器。**它基本上可以覆盖大部分基于长度拆包场景，开源消息中间件 RocketMQ 就是使用 LengthFieldBasedFrameDecoder 进行解码的



## 十三 writeAndFlush处理流程（Netty如何发送数据）

例如

![img](https://s0.lgstatic.com/i/image/M00/6D/AE/CgqCHl-uZz2AD58hAAHvDJLyzyU332.png)

```java
public class RequestSampleHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        String data = ((ByteBuf) msg).toString(CharsetUtil.UTF_8);
        ResponseSample response = new ResponseSample("OK", data, System.currentTimeMillis());
        ctx.channel().writeAndFlush(response);
    }
}
```

当 RequestSampleHandler 调用 writeAndFlush 时，数据是如何在 Pipeline 中传播、处理并向客户端发送的呢？

既然 writeAndFlush 是特有的出站操作，那么我们猜测它是从 Pipeline 的 Tail 节点开始传播的，然后一直向前传播到 Head 节点。

```java
@Override
public final ChannelFuture writeAndFlush(Object msg) {
    return tail.writeAndFlush(msg);
}
```

继续跟进 tail.writeAndFlush 的源码，最终会定位到 AbstractChannelHandlerContext 中的 write 方法。该方法是 writeAndFlush 的核心逻辑，

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    // ...... 省略部分非核心代码 ......

    // 找到 Pipeline 链表中下一个 Outbound 类型的 ChannelHandler 节点
    final AbstractChannelHandlerContext next = findContextOutbound(flush ?
            (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    // 判断当前线程是否是 NioEventLoop 中的线程
    if (executor.inEventLoop()) {
        if (flush) {
            // 因为 flush == true，所以流程走到这里
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        final AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        if (!safeExecute(executor, task, promise, m)) {
            task.cancel();
        }
    }
}
```

第一步，调用 findContextOutbound 方法找到 Pipeline 链表中下一个 Outbound 类型的 ChannelHandler。在我们模拟的场景中下一个 Outbound 节点是 ResponseSampleEncoder。

第二步，通过 inEventLoop 方法判断当前线程的身份标识，如果当前线程和 EventLoop 分配给当前 Channel 的线程是同一个线程的话，那么所提交的任务将被立即执行。否则当前的操作将被封装成一个 Task 放入到 EventLoop 的任务队列，稍后执行。 writeAndFlush 是线程安全的

第三步，因为 flush== true，将会直接执行 next.invokeWriteAndFlush(m, promise) 这行代码，我们跟进去源码。发现最终会它会执行下一个 ChannelHandler 节点的 write 方法，那么流程又回到了 到 AbstractChannelHandlerContext 中重复执行 write 方法，继续寻找下一个 Outbound 节点。

```java
private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
        invokeFlush0();
    } else {
        writeAndFlush(msg, promise);
    }
}
private void invokeWrite0(Object msg, ChannelPromise promise) {
    try {
        ((ChannelOutboundHandler) handler()).write(this, msg, promise);
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
```

调用 writeAndFlush 时数据是在 Outbound 类型的 ChannelHandler 节点之间进行传播，

数据将会在 Pipeline 中一直寻找 Outbound 节点并向前传播，直到 Head 节点结束，由 Head 节点完成最后的数据发送。

```java
// HeadContext # write
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    unsafe.write(msg, promise);
}
// AbstractChannel # AbstractUnsafe # write
@Override
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        safeSetFailure(promise, newClosedChannelException(initialCloseCause));
        ReferenceCountUtil.release(msg);
        return;
    }
    int size;
    try {
        msg = filterOutboundMessage(msg); // 过滤消息
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }
    outboundBuffer.addMessage(msg, size, promise); // 向 Buffer 中添加数据
}
```

write 方法还需要 ChannelPromise 参数，可见写操作是个异步的过程。AbstractChannelHandlerContext 会默认初始化一个 ChannelPromise 完成该异步操作，ChannelPromise 内部持有当前的 Channel 和 EventLoop，此外你可以向 ChannelPromise 中注册回调监听 listener 来获得异步操作的结果。

Head 节点是通过调用 unsafe 对象完成数据写入的，unsafe 对应的是 NioSocketChannelUnsafe 对象实例，最终调用到 AbstractChannel 中的 write 方法，该方法有两个重要的点需要指出：

1. filterOutboundMessage 方法会对待写入的 msg 进行过滤，如果 msg 使用的不是 DirectByteBuf，那么它会将 msg 转换成 DirectByteBuf。


2. ChannelOutboundBuffer 可以理解为一个缓存结构，从源码最后一行 outboundBuffer.addMessage 可以看出是在向这个缓存中添加数据，所以 ChannelOutboundBuffer 才是理解数据发送的关键。

#### writeAndFlush 是如何触发事件传播的？数据是怎样写到 Socket 底层的？

writeAndFlush 属于出站操作，它是从 Pipeline 的 Tail 节点开始进行事件传播，一直向前传播到 Head 节点。

在写时有两个操作，一个是write ，一个是flush

write操作是将数据写入ChannelOutboundBuffer ，需要判断缓存大小是否超过所设置的高水位线 64KB，如果超过了高水位，那么 Channel 会被设置为不可写状态。直到缓存的数据大小低于低水位线 32KB 以后，Channel 才恢复成可写状态。

flush操作是负责将数据真正写入到 Socket 缓冲区

#### 为什么会有 write 和 flush 两个动作？执行 flush 之前数据是如何存储的？

writeAndFlush 主要分为两个步骤，write 和 flush。

##### 执行write操作

通过上面的分析可以看出只调用 write 方法，数据并不会被真正发送出去，而是存储在 ChannelOutboundBuffer 的缓存内。

ChannelOutboundBuffer 缓存是一个链表结构，每次传入的数据都会被封装成一个 Entry 对象添加到链表中。ChannelOutboundBuffer 包含三个非常重要的指针：第一个被写到缓冲区的节点 flushedEntry、第一个未被写到缓冲区的节点 unflushedEntry和最后一个节点 tailEntry。

在初始状态下这三个指针都指向 NULL，当我们每次调用 write 方法是，都会调用 addMessage 方法改变这三个指针的指向，可以参考下图理解指针的移动过程会更加形象。

<img src="https://s0.lgstatic.com/i/image/M00/6D/AE/CgqCHl-uZ1GADbu0AAMyHCydEjU371.png" alt="图片13.png" style="zoom: 67%;" />

第一次调用 write，因为链表里只有一个数据，所以 unflushedEntry 和 tailEntry 指针都指向第一个添加的数据 msg1。flushedEntry 指针在没有触发 flush 动作时会一直指向 NULL。

第二次调用 write，tailEntry 指针会指向新加入的 msg2，unflushedEntry 保持不变。

第 N 次调用 write，tailEntry 指针会不断指向新加入的 msgN，unflushedEntry 依然保持不变，unflushedEntry 和 tailEntry 指针之间的数据都是未写入 Socket 缓冲区的。

以上便是写 Buffer 队列写入数据的实现原理，但是我们不可能一直向缓存中写入数据，所以 addMessage 方法中每次写入数据后都会调用 incrementPendingOutboundBytes 方法判断缓存的水位线，具体源码如下。

```java
private static final int DEFAULT_LOW_WATER_MARK = 32 * 1024;
private static final int DEFAULT_HIGH_WATER_MARK = 64 * 1024;
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }

    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
    // 判断缓存大小是否超过高水位线
    if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
        setUnwritable(invokeLater);
    }
}
```

incrementPendingOutboundBytes 的逻辑非常简单，每次添加数据时都会累加数据的字节数，然后判断缓存大小是否超过所设置的高水位线 64KB，如果超过了高水位，那么 Channel 会被设置为不可写状态。直到缓存的数据大小低于低水位线 32KB 以后，Channel 才恢复成可写状态。

##### 执行flush操作

当执行完 write 写操作之后，invokeFlush0 会触发 flush 动作，与 write 方法类似，flush 方法同样会从 Tail 节点开始传播到 Head 节点。

flush 的核心逻辑主要分为两个步骤：addFlush 和 flush0，

```java
// HeadContext # flush
@Override
public void flush(ChannelHandlerContext ctx) {
    unsafe.flush();
}
// AbstractChannel # flush
@Override
public final void flush() {
    assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }
    outboundBuffer.addFlush();
    flush0();
}
```

addFlush 方法同样也会操作 ChannelOutboundBuffer 缓存数据。在执行 addFlush 方法时，缓存中的指针变化又是如何呢？

<img src="https://s0.lgstatic.com/i/image/M00/6D/AE/CgqCHl-uZ2CAFvXuAAJkYjAgb8A346.png" alt="图片14.png" style="zoom: 50%;" />

此时 flushedEntry 指针有所改变，变更为 unflushedEntry 指针所指向的数据，然后 unflushedEntry 指针指向 NULL，flushedEntry 指针指向的数据才会被真正发送到 Socket 缓冲区。

第二步负责发送数据的 flush0 方法。

```java
@Override
protected final void flush0() {
    if (!isFlushPending()) {
        super.flush0();
    }
}
// AbstractNioByteChannel # doWrite
@Override
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    int writeSpinCount = config().getWriteSpinCount();
    do {
        Object msg = in.current();
        if (msg == null) {
            clearOpWrite();
            return;
        }
        writeSpinCount -= doWriteInternal(in, msg);
    } while (writeSpinCount > 0);
    incompleteWrite(writeSpinCount < 0);
}
```

实际 flush0 的调用层次很深，但其实核心的逻辑在于 AbstractNioByteChannel 的 doWrite 方法，该方法负责将数据真正写入到 Socket 缓冲区。

时序图

![img](https://s0.lgstatic.com/i/image/M00/6D/A3/Ciqc1F-uZ4iAYZDxAAROuJN6ruk510.png)

#### writeAndFlush 是同步还是异步？它是线程安全的吗？

异步，线程安全





## 十四 Netty为什么使用堆外内存

### Java中使用堆外内存

在堆内存放的 DirectByteBuffer 对象并不大，仅仅包含堆外内存的地址、大小等属性，同时还会创建对应的 Cleaner 对象，通过 ByteBuffer 分配的堆外内存不需要手动回收，它可以被 JVM 自动回收。当堆内的 DirectByteBuffer 对象被 GC 回收时，Cleaner 就会用于回收对应的堆外内存。

使用Unsafe 类，通过反射获取 Unsafe 实例。获得 Unsafe 实例后，可以通过 allocateMemory 方法分配堆外内存，allocateMemory 方法返回的是内存地址。与 DirectByteBuffer 不同的是，Unsafe#allocateMemory 所分配的内存必须自己手动释放，否则会造成内存泄漏，这也是 Unsafe 不安全的体现

### <img src="https://s0.lgstatic.com/i/image/M00/6F/43/CgqCHl-060uANzIXAAK8c10kJxc818.png" alt="图片2.png" style="zoom: 50%;" />

### 堆外内存的回收

 DirectByteBuffer 对象有可能长时间存在于堆内内存，所以它很可能晋升到 JVM 的老年代，所以这时候 DirectByteBuffer 对象的回收需要依赖 Old GC 或者 Full GC才能触发清理。如果长时间没有 Old GC 或者 Full GC 执行，那么堆外内存即使不再使用，也会一直在占用内存不释放，很容易将机器的物理内存耗尽。

最好通过 JVM 参数 **-XX:MaxDirectMemorySize** 指定堆外内存的上限大小，当堆外内存的大小超过该阈值时，就会触发一次 Full GC 进行清理回收，如果在 Full GC 之后还是无法满足堆外内存的分配，那么程序将会抛出 OOM 异常。

DirectByteBuffer 在初始化时会创建一个 Cleaner 对象，它会负责堆外内存的回收工作，通过虚引用与JVM垃圾回收进行关联。（虚引用会接收到系统GC通知）PhantomReference 不能被单独使用，需要与引用队列 ReferenceQueue 联合使用。

其中 PhantomReference 是最不常用的一种引用方式，Cleaner 就属于 PhantomReference 的子类，

当初始化堆外内存时，内存中的对象引用情况如下图所示，first 是 Cleaner 类中的静态变量，Cleaner 对象在初始化时会加入 Cleaner 链表中。DirectByteBuffer 对象包含堆外内存的地址、大小以及 Cleaner 对象的引用，ReferenceQueue 用于保存需要回收的 Cleaner 对象。

<img src="https://s0.lgstatic.com/i/image/M00/6F/37/Ciqc1F-063GAc4TOAATJbR2Lmao239.png" alt="图片3.png" style="zoom: 50%;" />

当发生 GC 时，DirectByteBuffer对象被回收，内存中的对象引用情况发生了如下变化，cleaner的虚引用放到引用对列里等待被回收，后面才是回收堆外内存，

<img src="https://s0.lgstatic.com/i/image/M00/6F/37/Ciqc1F-063eAQ7AiAAPPC1-cL1I933.png" alt="图片4.png" style="zoom:50%;" />



此时 Cleaner 对象不再有任何引用关系，在下一次 GC 时（虚引用会接收一个系统GC通知），此时该 Cleaner 对象将被添加到 ReferenceQueue 中，并执行 clean() 方法。clean() 方法主要做两件事情：

1. 将 Cleaner 对象从 Cleaner 链表中移除；


2. 调用 **unsafe.freeMemory** 方法清理堆外内存。





## 十五 什么是Netty 数据传输载体 ByteBuf 

 JDK NIO 的ByteBuffer 包含以下四个基本属性：

mark：为某个读取过的关键位置做标记，方便回退到该位置；

position：当前读取的位置；

limit：buffer 中有效的数据长度大小；

capacity：初始化时的空间容量。

<img src="https://s0.lgstatic.com/i/image/M00/6F/FE/Ciqc1F-3ukmAImo_AAJEEbA2rts301.png" alt="Netty11(1).png" style="zoom: 50%;" />

### ByteBuffer缺点

第一，ByteBuffer 分配的长度是固定的，无法动态扩缩容，所以很难控制需要分配多大的容量。

第二，ByteBuffer 只能通过 position 获取当前可操作的位置，因为读写共用的 position 指针。

ByteBuffer 作为网络通信中高频使用的数据载体，显然不能够满足 Netty 的需求，Netty 重新实现了一个性能更高、易用性更强的 ByteBuf，

### ByteBuf内部结构



![Netty11（2）.png](https://s0.lgstatic.com/i/image/M00/70/0A/CgqCHl-3uraAAhvwAASZGuNRMtA960.png)

ByteBuf 包含三个指针：读指针 readerIndex、写指针 writeIndex、最大容量 maxCapacity。

优点：由此可见，Netty 重新设计的 ByteBuf 有效地区分了可读、可写以及可扩容数据，解决了 ByteBuffer **无法扩容以及读写模式切换**烦琐的缺陷。

ByteBuf 是基于引用计数设计的，它实现了 ReferenceCounted 接口，ByteBuf 的生命周期是由引用计数所管理。只要引用计数大于 0，表示 ByteBuf 还在被使用；当 ByteBuf 不再被其他对象所引用时，引用计数为 0，那么代表该对象可以被释放。

当新创建一个 ByteBuf 对象时，它的初始引用计数为 1，当 ByteBuf 调用 release() 后，引用计数减 1，所以不要误以为调用了 release() 就会保证 ByteBuf 对象一定会被回收。

引用计数对于 Netty 设计缓存池化有非常大的帮助，当引用计数为 0，该 ByteBuf 可以被放入到对象池中，避免每次使用 ByteBuf 都重复创建，对于实现高性能的内存管理有着很大的意义。

### ByteBuf 分类

**Heap/Direct** 就是堆内和堆外内存。Heap 指的是在 JVM 堆内分配，底层依赖的是字节数据；Direct 则是堆外内存，不受 JVM 限制，分配方式依赖 JDK 底层的 ByteBuffer。

**Pooled/Unpooled** 表示池化还是非池化内存。Pooled 是从预先分配好的内存中取出，使用完可以放回 ByteBuf 内存池，等待下一次分配。而 Unpooled 是直接调用系统 API 去申请内存，确保能够被 JVM GC 管理回收。

**Unsafe/非 Unsafe** 的区别在于操作方式是否安全。 Unsafe 表示每次调用 JDK 的 Unsafe 对象操作物理内存，依赖 offset + index 的方式操作数据。非 Unsafe 则不需要依赖 JDK 的 Unsafe 对象，直接通过数组下标的方式操作数据。



## 十六 Netty内存管理

Netty 中引入类似 jemalloc 的内存池管理技术

Netty 默认提供了池化对象的内存分配，使用完后归还到内存池。

Netty 在每个区域内又定义了更细粒度的内存分配单位，分别为 Chunk、Page、Subpage，我们将逐一对其进行介绍。

Chunk 是 Netty 向操作系统申请内存的单位，所有的内存分配操作也是基于 Chunk 完成的，Chunk 可以理解为 Page 的集合，每个 Chunk 默认大小为 16M。

Page 是 Chunk 用于管理内存的单位，Netty 中的 Page 的大小为 8K，不要与 Linux 中的内存页 Page 相混淆了。假如我们需要分配 64K 的内存，需要在 Chunk 中选取 8 个 Page 进行分配。

Subpage 负责 Page 内的内存分配，假如我们分配的内存大小远小于 Page，直接分配一个 Page 会造成严重的内存浪费，所以需要将 Page 划分为多个相同的子块进行分配，这里的子块就相当于 Subpage。

分四种内存规格管理内存，分别为 Tiny、Samll、Normal、Huge，PoolChunk 负责管理 8K 以上的内存分配，PoolSubpage 用于管理 8K 以下的内存分配。当申请内存大于 16M 时，不会经过内存池，直接分配。

Netty 内存池的设计思想知识点总结

- 设计了本地线程缓存机制 PoolThreadCache，用于提升内存分配时的并发性能。用于申请 Tiny、Samll、Normal 三种类型的内存时，会优先尝试从 PoolThreadCache 中分配。


- PoolChunk 使用伙伴算法管理 Page，以二叉树的数据结构实现，是整个内存池分配的核心所在。


- 每调用 PoolThreadCache 的 allocate() 方法到一定次数，会触发检查 PoolThreadCache 中缓存的使用频率，使用频率较低的内存块会被释放。


- 线程退出时，Netty 会回收该线程对应的所有内存。

### 在单线程或者多线程的场景下，如何高效地进行内存分配和回收？



### 如何减少内存碎片，提高内存的有效利用率？



## 十七 Recycler对象池

Recycler 是 Netty 提供的自定义实现的轻量级对象回收站，借助 Recycler 可以完成对象的获取和回收。Recycler 是 Netty 自己实现的对象池。



知识点总结：

- 对象池有两个重要的组成部分：Stack 和 WeakOrderQueue。


- 从 Recycler 获取对象时，优先从 Stack 中查找，如果 Stack 没有可用对象，会尝试从 WeakOrderQueue 迁移部分对象到 Stack 中。


- Recycler 回收对象时，分为同线程对象回收和异线程对象回收两种情况，同线程回收直接向 Stack 中添加对象，异线程回收向 WeakOrderQueue 中的 Link 添加对象。


- 对象回收都会控制回收速率，每 8 个对象会回收一个，其他的全部丢弃。



## 十八 Netty零拷贝技术

### 零拷贝体现在哪些地方？

传统 Linux 中零拷贝的工作原理。所谓零拷贝，就是在数据操作时，不需要将数据从一个内存位置拷贝到另外一个内存位置，这样可以减少一次内存拷贝的损耗，从而节省了 CPU 时钟周期和内存带宽。

模拟一个场景，从文件中读取数据，然后将数据传输到网络上，那么传统的数据拷贝过程会分为哪几个阶段呢？

<img src="https://s0.lgstatic.com/i/image/M00/80/0B/Ciqc1F_Qbz2AD4uMAARnlgeSFc4993.png" alt="Drawing 0.png" style="zoom:33%;" />

在 Linux 中系统调用 sendfile() 可以实现将数据从一个文件描述符传输到另一个文件描述符，从而实现了零拷贝技术。

 Java 中也使用了零拷贝技术，它就是 NIO FileChannel 类中的 transferTo() 方法，transferTo() 底层就依赖了操作系统零拷贝的机制，它可以将数据从 FileChannel 直接传输到另外一个 Channel。

<img src="https://s0.lgstatic.com/i/image/M00/80/16/CgqCHl_Qb0mANyjrAATEtVu9f6c390.png" alt="Drawing 1.png" style="zoom:33%;" />

DMA 引擎从文件中读取数据拷贝到内核态缓冲区之后，由操作系统直接拷贝到 Socket 缓冲区，不再拷贝到用户态缓冲区，所以数据拷贝的次数从之前的 4 次减少到 3 次。

### Netty 的零拷贝技术是如何实现的呢？

只要能够减少不必要的 CPU 拷贝，都可以被称为零拷贝。

Netty 中的零拷贝技术除了操作系统级别的功能封装，更多的是面向用户态的数据操作优化，主要体现在以下 5 个方面：

- 堆外内存，避免 JVM 堆内存到堆外内存的数据拷贝。

  如果在 JVM 内部执行 I/O 操作时，必须将数据拷贝到堆外内存，才能执行系统调用。

  Netty 在进行 I/O 操作时都是使用的堆外内存，可以避免数据从 JVM 堆内存到堆外内存的拷贝。

- CompositeByteBuf 类，可以组合多个 Buffer 对象合并成一个逻辑上的对象，避免通过传统内存拷贝的方式将几个 Buffer 合并成一个大的 Buffer。

- 通过 Unpooled.wrappedBuffer 可以将 byte 数组包装成 ByteBuf 对象，包装过程中不会产生内存拷贝。

  Unpooled.wrappedBuffer 方法可以将不同的数据源的一个或者多个数据包装成一个大的 ByteBuf 对象，其中数据源的类型包括 byte[]、ByteBuf、ByteBuffer。包装的过程中不会发生数据拷贝操作，包装后生成的 ByteBuf 对象和原始 ByteBuf 对象是共享底层的 byte 数组。

- ByteBuf.slice 操作与 Unpooled.wrappedBuffer 相反，slice 操作可以将一个 ByteBuf 对象切分成多个 ByteBuf 对象，切分过程中不会产生内存拷贝，底层共享一个 byte 数组的存储空间。

- Netty 使用 FileRegion 实现文件传输，FileRegion 底层封装了 FileChannel#transferTo() 方法，可以将文件缓冲区的数据直接传输到目标 Channel，避免内核缓冲区和用户态缓冲区之间的数据拷贝，这属于操作系统级别的零拷贝。

  





## 十九 Netty服务端启动流程

在服务端启动之前，需要配置 ServerBootstrap 的相关参数，

- 配置 EventLoopGroup 线程组；
- 配置 Channel 的类型；
- 设置 ServerSocketChannel 对应的 Handler；
- 设置网络监听的端口；
- 设置 SocketChannel 对应的 Handler；
- 配置 Channel 参数。

```java
public class EchoServer {
    public void startEchoServer(int port) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler(LogLevel.INFO)) // 设置ServerSocketChannel 对应的 Handler
                    .childHandler(new ChannelInitializer<SocketChannel>() { // 设置 SocketChannel 对应的 Handler
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(new FixedLengthFrameDecoder(10));
                            ch.pipeline().addLast(new ResponseSampleEncoder());
                            ch.pipeline().addLast(new RequestSampleHandler());
                        }
                    });
            ChannelFuture f = b.bind(port).sync();
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

![img](https://s0.lgstatic.com/i/image/M00/87/A2/CgqCHl_WABCAJA9VAAIG0Ncq3hs061.png)

参数配置完成，执行bind()方法，创建初始化启动服务端，为服务端启动入口方法。主要完成以下四步

- 创建服务端 Channel

  > Netty的中的Channel实现类主要有：
  >
  > NioServerSocketChannel（用于服务端非阻塞地接收TCP连接）、
  >
  > NioSocketChannel（用于维持非阻塞的TCP连接）、
  >
  > NioDatagramChannel（用于非阻塞地处理UDP连接）、
  >
  > OioServerSocketChannel（用于服务端阻塞地接收TCP连接）、
  >
  > OioSocketChannel（用于阻塞地接收TCP连接）、
  >
  > OioDatagramChannel（用于阻塞地处理UDP连接）：

  **通过反射创建 NioServerSocketChannel 实例；创建 JDK 底层的 ServerSocketChannel；**

  通过 channel(NioServerSocketChannel.class) 配置 Channel 的类型，在 Linux 内核 2.6版本及以上都会默认采用 EPollSelectorProvider。如果是旧版或者windows使用PollSelectorProvider。

- 初始化服务端 Channel

  **设置 Socket 参数以及用户自定义属性，并添加两个特殊的处理器 ChannelInitializer 和 ServerBootstrapAcceptor。从 ServerBootstrapAcceptor 的命名可以看出，这是一个连接接入器，专门用于接收新的连接，然后把事件分发给 EventLoop 执行，**

- 注册服务端 Channel

  **调用 JDK 底层将 Channel 注册到 Selector 上。**

  Netty 会在线程池 EventLoopGroup 中选择一个 EventLoop 与当前 Channel 进行绑定，之后 Channel 生命周期内的所有 I/O 事件都由这个 EventLoop 负责处理，如 accept、connect、read、write 等 I/O 事件。通过resgister()方法进行注册

  register0() 主要做了四件事：调用 JDK 底层进行 Channel 注册、触发 handlerAdded 事件、触发 channelRegistered 事件、Channel 当前状态为活跃时，触发 channelActive 事件。

  同时会调用JDK 底层注册 Channel ，将 Channel 注册到 Selector 上，register() 的第三个入参传入的是 Netty 自己实现的 Channel 对象，调用 register() 方法会将它绑定在 JDK 底层 Channel 的 attachment 上。这样在每次 Selector 对象进行事件循环时，Netty 都可以从返回的 JDK 底层 Channel 中获得自己的 Channel 对象。

  完成 Channel 向 Selector 注册后，接下来就会触发 Pipeline 一系列的事件传播。

- 端口绑定

  **bind() 方法主要做了两件事，分别为调用 JDK 底层进行端口绑定；绑定成功后并触发 channelActive 事件，把 OP_ACCEPT 事件注册到 Channel 的事件集合中。**



## 二十 Netty 服务端是如何处理客户端新建连接的呢？

Netty 服务端完全启动后，如何处理客户端新建连接？

1. Boss NioEventLoop 线程轮询客户端新连接 OP_ACCEPT 事件；

   Netty 中 Boss NioEventLoop 专门负责接收新的连接，

2. 构造 Netty 客户端 NioSocketChannel；

   Netty 先通过 JDK 底层的 accept() 获取 JDK 原生的 SocketChannel，然后将它封装成 Netty 自己的 NioSocketChannel。

   成功构造客户端 NioSocketChannel 后，接下来会通过 pipeline.fireChannelRead() 触发 channelRead 事件传播。

3. 注册 Netty 客户端 NioSocketChannel 到 Worker 工作线程中；

   channelRead() 会将客户端 Channel 分配到工作线程组中去执行

4. 注册 OP_READ 事件到 NioSocketChannel 的事件集合。

   注册 OP_READ 事件到 NioSocketChannel 的事件集合。

   注册过程会调用 pipeline.fireChannelRegistered() 方法传播 channelRegistered 事件，

   然后再调用 pipeline.fireChannelActive() 方法传播 channelActive 事件



Netty 服务端启动后，BossEventLoopGroup 会负责监听客户端的 Accept 事件。当有客户端新连接接入时，BossEventLoopGroup 中的 NioEventLoop 首先会新建客户端 Channel，然后在 NioServerSocketChannel 中触发 channelRead 事件传播，NioServerSocketChannel 中包含了一种特殊的处理器 ServerBootstrapAcceptor，最终通过 ServerBootstrapAcceptor 的 channelRead() 方法将新建的客户端 Channel 分配到 WorkerEventLoopGroup 中。WorkerEventLoopGroup 中包含多个 NioEventLoop，它会选择其中一个 NioEventLoop 与新建的客户端 Channel 绑定。

完成客户端连接注册之后，就可以接收客户端的请求数据了。当客户端向服务端发送数据时，NioEventLoop 会监听到 OP_READ 事件，然后分配 ByteBuf 并读取数据，读取完成后将数据传递给 Pipeline 进行处理。一般来说，数据会从 ChannelPipeline 的第一个 ChannelHandler 开始传播，将加工处理后的消息传递给下一个 ChannelHandler，整个过程是串行化执行。





## 二十一 Reactor 线程模型，EventLoop线程执行主要流程

NioEventLoop 的 run() 方法是一个无限循环，没有任何退出条件，在不间断循环执行以下三件事情，

<img src="https://s0.lgstatic.com/i/image2/M01/02/1E/Cip5yF_ZyhCAM3GUAASrdEcuR2U593.png" alt="Lark20201216-164824.png" style="zoom: 50%;" />



- 轮询 I/O 事件（select）：轮询 Selector 选择器中已经注册的所有 Channel 的 I/O 事件。

  NioEventLoop 通过核心方法 select() 不断轮询注册的 I/O 事件。当没有 I/O 事件产生时，为了避免 NioEventLoop 线程一直循环空转，在获取 I/O 事件或者异步任务时需要阻塞线程，等待 I/O 事件就绪或者异步任务产生后才唤醒线程。NioEventLoop 使用 wakeUp 变量表示是否唤醒 selector，Netty 在每一次执行新的一轮循环之前，都会将 wakeUp 设置为 false。

  Netty 解决 JDK epoll 空轮询 Bug

  1. 每次执行 Select 操作之前记录当前时间 currentTimeNanos。
  2. time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos，如果事件轮询的持续时间大于等于 timeoutMillis，那么说明是正常的，否则表明阻塞时间并未达到预期，可能触发了空轮询的 Bug。
  3. Netty 引入了计数变量 selectCnt。在正常情况下，selectCnt 会重置，否则会对 selectCnt 自增计数。当 selectCnt 达到 SELECTOR_AUTO_REBUILD_THRESHOLD（默认512） 阈值时，会触发重建 Selector 对象。

  简单总结 select 过程所做的事情。select 操作也是一个无限循环，在事件轮询之前检查任务队列是否为空，确保任务队列中待执行的任务能够及时执行。如果任务队列中已经为空，然后执行 select 阻塞操作获取等待获取 I/O 事件。Netty 通过引入计数器变量，并统计在一定时间窗口内 select 操作的执行次数，识别出可能存在异常的 Selector 对象，然后采用重建 Selector 的方式巧妙地避免了 JDK epoll 空轮询的问题。

- 处理 I/O 事件（processSelectedKeys）：处理已经准备就绪的 I/O 事件。

  通过 select 过程我们已经获取到准备就绪的 I/O 事件，接下来就需要调用 processSelectedKeys() 方法处理 I/O 事件。

  processSelectedKey 一共处理了 OP_CONNECT、OP_WRITE、OP_READ 三个事件，我们分别了解下这三个事件的处理过程

- 处理异步任务队列（runAllTasks）：Reactor 线程还有一个非常重要的职责，就是处理任务队列中的非 I/O 任务。Netty 提供了 ioRatio 参数用于调整 I/O 事件处理和任务处理的时间比例。

  NioEventLoop 内部有两个非常重要的异步任务队列，分别为普通任务队列和定时任务队列。NioEventLoop 提供了 execute() 和 schedule() 方法用于向不同的队列中添加任务，execute() 用于添加普通任务，schedule() 方法用于添加定时任务。



## 二十二 Netty中的FastThreadLocal 









## 二十三 Netty 中队列Mpsc Queue 

Mpsc 的全称是 Multi Producer Single Consumer，多生产者单消费者。Mpsc Queue 可以保证多个生产者同时访问队列是线程安全的，而且同一时刻只允许一个消费者从队列中读取数据。Netty Reactor 线程中任务队列 taskQueue 必须满足多个生产者可以同时提交任务。





## 二十四 什么是select、poll、epoll？

### select 

时间复杂度O(n)

有I/O事件发生了，却并不知道是哪那几个流（可能有一个，多个，甚至全部），只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。所以select具有O(n)的无差别轮询复杂度，同时处理的流越多，无差别轮询时间就越长。

select本质上是通过设置或者检查存放fd标志位的数据结构来进行下一步处理。

#### 实现



#### 缺点

**（1）每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大**,select低效的另一个原因在于程序不知道哪些socket收到数据，只能一个个遍历。

**（2）同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大**

**（3）select支持的文件描述符数量太小了，默认是1024**

### poll

时间复杂度O(n)

poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd（文件描述符）对应的设备状态，如果设备就绪则在设备等待队列中加入一项并继续遍历，如果遍历完所有fd后没有发现就绪设备，则挂起当前进程，直到设备就绪或者主动超时，被唤醒后它又要再次遍历fd

大量的fd的数组被整体复制于用户态和内核地址空间之间，

**poll没有最大文件描述符数量的限制**

### epoll

epoll可以理解为event poll，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们。所以说epoll实际上是事件驱动（每个事件关联上fd）的。

epoll有EPOLLLT和EPOLLET两种触发模式，LT是默认的模式，ET是“高速”模式。LT模式下，只要这个fd还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作，而在ET（边缘触发）模式中，它只会提示一次，直到下次再有数据流入之前都不会再提示了，无 论fd中是否还有数据可读。所以在ET模式下，read一个fd的时候一定要把它的buffer读光，也就是说一直读到read的返回值小于请求值，或者 遇到EAGAIN错误。还有一个特点是，epoll使用“事件”的就绪通知方式，通过epoll_ctl注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制来激活该fd，epoll_wait便可以收到通知。

#### 实现

epoll提供了三个函数，epoll_create，epoll_ctl和epoll_wait，

epoll_create是创建一个epoll句柄；

epoll_ctl是将需要监视的socket添加到epfd中

epoll_wait则是等待事件的产生。

#### 流程

**创建epoll对象**

调用epoll_create方法时，内核会创建一个eventpoll对象（也就是程序中epfd所代表的对象）

**维护监视列表**

创建epoll对象后，可以用epoll_ctl添加或删除所要监听的socket。以添加socket为例，如下图，如果通过epoll_ctl添加sock1、sock2和sock3的监视，内核会将eventpoll添加到这三个socket的等待队列中。

**添加所要监听的socket**

当socket收到数据后，中断程序会操作eventpoll对象，而不是直接操作进程（也就是调用epoll的进程）。

**接收数据**

当socket收到数据后，中断程序会给eventpoll的“就绪列表”添加socket引用。

给就绪列表添加引用

eventpoll对象相当于是socket和进程之间的中介，socket的数据接收并不直接影响进程，而是通过改变eventpoll的就绪列表来改变进程状态。

当程序执行到epoll_wait时，如果rdlist已经引用了socket，那么epoll_wait直接返回，如果rdlist为空，阻塞进程。

**epoll_wait阻塞进程**

当socket接收到数据，中断程序一方面修改rdlist，另一方面唤醒eventpoll等待队列中的进程，进程A再次进入运行状态（如下图）。也因为rdlist的存在，进程A可以知道哪些socket发生了变化。





　　对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。

　　对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。

　　对于第三个缺点，epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。



#### epoll的优点

1、没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）；
2、效率提升，不是轮询的方式，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数；
即Epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，Epoll的效率就会远远高于select和poll。

3、 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销。

**select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的**，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。 



### 总结

（1）select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。

（2）select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。 



Netty可以选择使用不同的多路复用技术。

NioEventLoop底层会根据系统选择select或者epoll。如果是windows系统，则底层使用WindowsSelectorProvider（select）实现多路复用；如果是linux，则使用epoll。

NioEventLoop在运行的时候，会不断的监听注册的连接，核心逻辑在doSelect方法中，主要的几个操作是
（1）processUpdateQueue，将newKeys，updateKeys，cancelledKeys中的key更新到PollArrayWrapper中
（2）subSelector.poll()，这里实际是调用native方法，会将PollArrayWrapper的fd拷贝至readFds/writeFds/exceptFds，并监听这些fd。
（3）updateSelectedKeys，遍历freadFds/writeFds/exceptFds，将就绪的fd存储到selectedKey中

当存在fd就绪后，doSelect方法返回，应用程序可以遍历selectedKey进行处理。



### 参考

https://zhuanlan.zhihu.com/p/63179839

https://blog.csdn.net/songchuwang1868/article/details/89877739

https://www.cnblogs.com/aspirant/p/9166944.html

## 二十五 Netty中使用了哪些设计模式

### 单例模式

#### 双重检验锁

```java
public class SingletonTest {
    private volatile static SingletonTest instance;
    
    public SingletonTest getInstance() {
        if (instance == null) {
            synchronized (this) {
                if (instance == null) {
                    instance = new SingletonTest();
                }
            }
        }
        return instance;
    }
}
```

#### 静态内部类方式

静态内部类方式实现单例巧妙地利用了 Java 类加载机制，保证其在多线程环境下的线程安全性。当一个类被加载时，其静态内部类是不会被同时加载的，只有第一次被调用时才会初始化，而且我们不能通过反射的方式获取内部的属性。由此可见，静态内部类方式实现单例更加安全，可以防止被反射入侵。

```java
public class SingletonTest {
    private SingletonTest() {
    }
    public static Singleton getInstance() {
        return SingletonInstance.instance;
    }
    private static class SingletonInstance {
        private static final Singleton instance = new Singleton();
    }
}
```

#### 饿汉方式

饿汉式实现单例非常简单，类加载的时候就创建出实例。饿汉方式使用私有构造函数实现全局单个实例的初始化，并使用 public static final 加以修饰，实现延迟加载和保证线程安全性。

```java
public class SingletonTest {
    private static Singleton instance = new Singleton();
    private Singleton() {
    }
    public static Singleton getInstance() {
        return instance;
    }
}
```

实际应用

1. NioEventLoop 通过核心方法 select() 不断轮询注册的 I/O 事件，Netty 提供了选择策略 SelectStrategy 对象，它用于控制 select 循环行为，包含 CONTINUE、SELECT、BUSY_WAIT 三种策略。SelectStrategy 对象的默认实现就是使用的饿汉式单例。



#### 枚举方式

枚举方式是一种天然的单例实现，能够保证序列化和反序列化过程中实例的唯一性，而且不用担心线程安全问题。

```java

```



### 工厂方法模式

工厂模式封装了对象创建的过程，使用者不需要关心对象创建的细节。工厂模式分为三种：**简单工厂模式**、**工厂方法模式**和**抽象工厂模式**。

- **简单工厂模式**。定义一个工厂类，根据参数类型返回不同类型的实例。适用于对象实例类型不多的场景，如果对象实例类型太多，每增加一种类型就要在工厂类中增加相应的创建逻辑，这是违背开放封闭原则的。
- **工厂方法模式**。简单工厂模式的升级版，不再是提供一个统一的工厂类来创建所有对象的实例，而是每种类型的对象实例都对应不同的工厂类，每个具体的工厂类只能创建一个类型的对象实例。
- **抽象工厂模式**。较少使用，适用于创建多个产品的场景。如果按照工厂方法模式的实现思路，需要在具体工厂类中实现多个工厂方法，是非常不友好的。抽象工厂模式就是把这些工厂方法单独剥离到抽象工厂类中，然后创建工厂对象并通过组合的方式来获取工厂方法。

实际应用场景

1. Netty 在创建 Channel 的时候使用的就是工厂方法模式，因为服务端和客户端的 Channel (NioServerSocketChannel、NioServerChannel)是不一样的。Netty 将反射和工厂方法模式结合在一起，只使用一个工厂类，然后根据传入的 Class 参数来构建出对应的 Channel，不需要再为每一种 Channel 类型创建一个工厂类。

   ```java
   public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {
       private final Constructor<? extends T> constructor;
       public ReflectiveChannelFactory(Class<? extends T> clazz) {
           ObjectUtil.checkNotNull(clazz, "clazz");
           try {
               this.constructor = clazz.getConstructor();
           } catch (NoSuchMethodException e) {
               throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                       " does not have a public non-arg constructor", e);
           }
       }
       @Override
       public T newChannel() {
           try {
               return constructor.newInstance();
           } catch (Throwable t) {
               throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
           }
       }
       @Override
       public String toString() {
           return StringUtil.simpleClassName(ReflectiveChannelFactory.class) +
                   '(' + StringUtil.simpleClassName(constructor.getDeclaringClass()) + ".class)";
       }
   }
   ```

   

### 责任链模式

用来处理相关事务责任的一条执行链，执行链上有多个节点，每个节点都有机会（条件匹配）处理请求事务，如果某个节点处理完了就可以根据实际业务需求传递给下一个节点继续处理或者返回处理完毕。

需要定义责任处理接口，责任链创建实现该接口

ChannlPipeline 和 ChannelHandler。ChannlPipeline 内部是由一组 ChannelHandler 实例组成的，内部通过双向链表将不同的 ChannelHandler 链接在一起。

![Drawing 0.png](https://s0.lgstatic.com/i/image2/M01/08/2F/Cip5yGAKfLmANJ_0AAZUvQP4FxQ293.png)

#### 责任处理器接口

ChannelHandler 对应的就是责任处理器接口，ChannelHandler 有两个重要的子接口：ChannelInboundHandler和ChannelOutboundHandler，分别拦截入站和出站的各种 I/O 事件。

#### 动态创建责任链，添加、删除责任处理器

ChannelPipeline 负责创建责任链，其内部采用双向链表实现，

```java
public class DefaultChannelPipeline implements ChannelPipeline {
    static final InternalLogger logger = InternalLoggerFactory.getInstance(DefaultChannelPipeline.class);
    private static final String HEAD_NAME = generateName0(HeadContext.class);
    private static final String TAIL_NAME = generateName0(TailContext.class);
    // 省略其他代码
    final AbstractChannelHandlerContext head; // 头结点
    final AbstractChannelHandlerContext tail; // 尾节点
    private final Channel channel;
    private final ChannelFuture succeededFuture;
    private final VoidChannelPromise voidPromise;
    private final boolean touch = ResourceLeakDetector.isEnabled();

    // 省略其他代码
```

ChannelPipeline 提供了一系列 add 和 remove 相关接口用于动态添加和删除 ChannelHandler 处理器。

#### 上下文

从 ChannelPipeline 内部结构定义可以看出，ChannelHandlerContext 负责保存责任链节点上下文信息。ChannelHandlerContext 是对 ChannelHandler 的封装，每个 ChannelHandler 都对应一个 ChannelHandlerContext，实际上 ChannelPipeline 维护的是与 ChannelHandlerContext 的关系。

#### 责任传播和终止机制

ChannelHandlerContext 提供了 fire 系列的方法用于事件传播

<img src="https://s0.lgstatic.com/i/image2/M01/08/30/Cip5yGAKfM6AHTnGAA43hnlCx54417.png" alt="Drawing 2.png" style="zoom:50%;" />



以 ChannelInboundHandlerAdapter 的 channelRead 方法为例，ChannelHandlerContext 会默认调用 fireChannelRead 方法将事件默认传递到下一个处理器。如果我们重写了 ChannelInboundHandlerAdapter 的 channelRead 方法，并且没有调用 fireChannelRead 进行事件传播，那么表示此次事件传播已终止。

### 观察者模式

观察者模式有两个角色：观察者和被观察。被观察者发布消息，观察者订阅消息，没有订阅的观察者是收不到消息的。

```java
// 被观察者
public interface Observable {
    void registerObserver(Observer observer);
    void removeObserver(Observer observer);
    void notifyObservers(String message);
}
// 观察者
public interface Observer {
    void notify(String message);
}
// 默认被观察者实现
public class DefaultObservable implements Observable {
    private final List<Observer> observers = new ArrayList<>();
    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }
    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }
    @Override
    public void notifyObservers(String message) {
        for (Observer observer : observers) {
            observer.notify(message);
        }
    }
}
```

#### 实际应用







### 建造者模式

当一个类的构造函数参数个数超过4个，而且这些参数有些是可选的参数，考虑使用构造者模式。

通过链式调用来设置对象的属性，在对象属性繁多的场景下非常有用。建造者模式的优势就是可以像搭积木一样自由选择需要的属性，并不是强绑定的。对于使用者来说，必须清楚需要设置哪些属性，在不同场景下可能需要的属性也是不一样的。



#### 实现

builder模式有4个角色。

- Product: 最终要生成的对象，例如 Computer实例。

- Builder： 构建者的抽象基类（有时会使用接口代替）。其定义了构建Product的抽象步骤，其实体类需要实现这些步骤。其会包含一个用来返回最终产品的方法`Product getProduct()`。
- ConcreteBuilder: Builder的实现类。
- Director: 决定如何构建最终产品的算法. 其会包含一个负责组装的方法`void Construct(Builder builder)`， 在这个方法中通过调用builder的方法，就可以设置builder，等设置完成后，就可以通过builder的 `getProduct()` 方法获得最终的产品。

> 第一步：我们的目标Computer类：
>
> ```text
> public class Computer {
>     private String cpu;//必须
>     private String ram;//必须
>     private int usbCount;//可选
>     private String keyboard;//可选
>     private String display;//可选
> 
>     public Computer(String cpu, String ram) {
>         this.cpu = cpu;
>         this.ram = ram;
>     }
>     public void setUsbCount(int usbCount) {
>         this.usbCount = usbCount;
>     }
>     public void setKeyboard(String keyboard) {
>         this.keyboard = keyboard;
>     }
>     public void setDisplay(String display) {
>         this.display = display;
>     }
>     @Override
>     public String toString() {
>         return "Computer{" +
>                 "cpu='" + cpu + '\'' +
>                 ", ram='" + ram + '\'' +
>                 ", usbCount=" + usbCount +
>                 ", keyboard='" + keyboard + '\'' +
>                 ", display='" + display + '\'' +
>                 '}';
>     }
> }
> ```
>
> 第二步：抽象构建者类
>
> ```text
> public abstract class ComputerBuilder {
>     public abstract void setUsbCount();
>     public abstract void setKeyboard();
>     public abstract void setDisplay();
> 
>     public abstract Computer getComputer();
> }
> ```
>
> 第三步：实体构建者类，我们可以根据要构建的产品种类产生多了实体构建者类，这里我们需要构建两种品牌的电脑，苹果电脑和联想电脑，所以我们生成了两个实体构建者类。
>
> 苹果电脑构建者类
>
> ```text
> public class MacComputerBuilder extends ComputerBuilder {
>     private Computer computer;
>     public MacComputerBuilder(String cpu, String ram) {
>         computer = new Computer(cpu, ram);
>     }
>     @Override
>     public void setUsbCount() {
>         computer.setUsbCount(2);
>     }
>     @Override
>     public void setKeyboard() {
>         computer.setKeyboard("苹果键盘");
>     }
>     @Override
>     public void setDisplay() {
>         computer.setDisplay("苹果显示器");
>     }
>     @Override
>     public Computer getComputer() {
>         return computer;
>     }
> }
> ```
>
> 联想电脑构建者类
>
> ```text
> public class LenovoComputerBuilder extends ComputerBuilder {
>     private Computer computer;
>     public LenovoComputerBuilder(String cpu, String ram) {
>         computer=new Computer(cpu,ram);
>     }
>     @Override
>     public void setUsbCount() {
>         computer.setUsbCount(4);
>     }
>     @Override
>     public void setKeyboard() {
>         computer.setKeyboard("联想键盘");
>     }
>     @Override
>     public void setDisplay() {
>         computer.setDisplay("联想显示器");
>     }
>     @Override
>     public Computer getComputer() {
>         return computer;
>     }
> }
> ```
>
> 第四步：指导者类（Director）
>
> ```text
> public class ComputerDirector {
>     public void makeComputer(ComputerBuilder builder){
>         builder.setUsbCount();
>         builder.setDisplay();
>         builder.setKeyboard();
>     }
> }
> ```
>
> ## 使用
>
> 首先生成一个director (1)，然后生成一个目标builder (2)，接着使用director组装builder (3),组装完毕后使用builder创建产品实例 (4)。
>
> ```text
> public static void main(String[] args) {
>         ComputerDirector director=new ComputerDirector();//1
>         ComputerBuilder builder=new MacComputerBuilder("I5处理器","三星125");//2
>         director.makeComputer(builder);//3
>         Computer macComputer=builder.getComputer();//4
>         System.out.println("mac computer:"+macComputer.toString());
> 
>         ComputerBuilder lenovoBuilder=new LenovoComputerBuilder("I7处理器","海力士222");
>         director.makeComputer(lenovoBuilder);
>         Computer lenovoComputer=lenovoBuilder.getComputer();
>         System.out.println("lenovo computer:"+lenovoComputer.toString());
> }
> ```
>
> 输出结果如下：
>
> ```text
> mac computer:Computer{cpu='I5处理器', ram='三星125', usbCount=2, keyboard='苹果键盘', display='苹果显示器'}
> lenovo computer:Computer{cpu='I7处理器', ram='海力士222', usbCount=4, keyboard='联想键盘', display='联想显示
> ```

#### 实际应用

Netty 中 ServerBootStrap 和 Bootstrap 引导器是最经典的建造者模式实现，在构建过程中需要设置非常多的参数，例如配置线程池 EventLoopGroup、设置 Channel 类型、注册 ChannelHandler、设置 Channel 参数、端口绑定等。



### 策略模式

策略模式针对同一个问题提供多种策略的处理方式，这些策略之间可以相互替换，在一定程度上提高了系统的灵活性。策略模式非常符合开闭原则，使用者在不修改现有系统的情况下选择不同的策略，而且便于扩展增加新的策略。

#### 实际应用

Netty 在多处地方使用了策略模式，例如 EventExecutorChooser 提供了不同的策略选择 NioEventLoop，newChooser() 方法会根据线程池的大小是否是 2 的幂次，以此来动态的选择取模运算的方式，从而提高性能。EventExecutorChooser 源码实现如下所示：

```java
public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory {
    public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();
    private DefaultEventExecutorChooserFactory() { }
    @SuppressWarnings("unchecked")
    @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }

    // 省略其他代码
}
```



### 装饰者模式

装饰器模式是对被装饰类的功能增强，在不修改被装饰类的前提下，能够为被装饰类添加新的功能特性。当我们需要为一个类扩展功能时会使用装饰器模式，但是该模式的缺点是需要增加额外的代码。

#### 实现

```java
public interface Shape {
    void draw();
}
class Circle implements Shape {
    @Override
    public void draw() {
        System.out.print("draw a circle.");
    }
}
abstract class ShapeDecorator implements Shape {
    protected Shape shapeDecorated;
    public ShapeDecorator(Shape shapeDecorated) {
        this.shapeDecorated = shapeDecorated;
    }
    public void draw() {
        shapeDecorated.draw();
    }
}
class FillReadColorShapeDecorator extends ShapeDecorator {
    public FillReadColorShapeDecorator(Shape shapeDecorated) {
        super(shapeDecorated);
    }
    @Override
    public void draw() {
        shapeDecorated.draw();
        fillColor();
    }
    private void fillColor() {
        System.out.println("Fill Read Color.");
    }
}
```

创建了一个 Shape 接口的抽象装饰类 ShapeDecorator，并维护 Shape 原始对象，FillReadColorShapeDecorator 是用于装饰 ShapeDecorator 的实体类，它不对 draw() 方法做任何修改，而是直接调用 Shape 对象原有的 draw() 方法，然后再调用 fillColor() 方法进行颜色填充。

#### 实际应用

 Netty 中 WrappedByteBuf 是如何装饰 ByteBuf 的，

```java
class WrappedByteBuf extends ByteBuf {
    protected final ByteBuf buf;
    protected WrappedByteBuf(ByteBuf buf) {
        if (buf == null) {
            throw new NullPointerException("buf");
        }
        this.buf = buf;
    }
    @Override
    public final boolean hasMemoryAddress() {
        return buf.hasMemoryAddress();
    }
    @Override
    public final long memoryAddress() {
        return buf.memoryAddress();
    }

    // 省略其他代码
}
```

WrappedByteBuf 是所有 ByteBuf 装饰器的基类，它并没有什么特别的，也是在构造函数里传入了原始的 ByteBuf 实例作为被装饰者。WrappedByteBuf 有两个子类 UnreleasableByteBuf 和 SimpleLeakAwareByteBuf，它们是真正实现对 ByteBuf 的功能增强，例如 UnreleasableByteBuf 类的 release() 方法是直接返回 false 表示不可被释放，源码实现如下所示。

不知道你会不会有一个疑问，装饰器模式和代理模式都是实现目标类增强，他们有什么区别吗？装饰器模式和代理模式的实现确实是非常相似的，都需要维护原始的目标对象，装饰器模式侧重于为目标类增加新的功能，代理模式更侧重于在现有功能的基础上进行扩展。

### 其它

- 模板方法模式：ServerBootStrap 和 Bootstrap 的 init 过程实现；
- 迭代器模式：CompositeByteBuf；
- 适配器模式：ScheduledFutureTask。





