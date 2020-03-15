## 一 线程安全

### 1. 并发与并行区别？





### 2.什么是线程安全？

多线程编程中，可能出现多个线程同时访问同一个共享、可变的资源（共享：多个线程能够同时访问；可变，资源能够被修改）。

多线程执行过程中，由于CPU时间片的轮转，线程执行过程并不完全可控，导致期望结果与实际不符，带来线程不安全，如果都能表现出正确地结果，那么即是线程安全。

其根源是由于计算机性能提升带来的内存可见性问题，原子性问题和有序性问题。

为了保证线程安全，需要保证线程同步，有序处理。

（下面为扩展）

Java中同步控制器包含synchronized和AQS。

- synchronized，基于JVM底层实现，不可控
- AQS，通过自旋、CAS、阻塞，实现的机制





## 二 Java内存模型

#### 2.1 什么是Java内存模型？

Java内存模型其实就是规定了工作内存与主内存交换数据的方式。

工作内存是指每个线程拥有的内存，物理上对应高速缓存，保存了该线程需要使用的变量在主内存中副本的拷贝。可对应于虚拟机栈中部分数据。线程私有，线程与线程之间变量的传递需要通过主内存完成。

主内存是虚拟机内存中的一部分，可对应于Java堆中的对象实例数据部分。

![工作内存与主内存交互](.\img\03\03_01.PNG)



#### 2.2 工作内存与主内存数据如何交互？

主要是通过6个指令，read、load、use、assign、store、write，和遵从缓存一致性协议，及CPU嗅探机制等。

![工作内存与主内存交互](.\img\03\03_02.PNG)



扩展：

| 操作名 | 作用                                                         |
| ------ | ------------------------------------------------------------ |
| lock   | 将主内存中一个变量标识为一条线程独占                         |
| unlock | 将主内存中一个被线程独占的变量释放出来                       |
| read   | 将主内存中一个变量传输到线程的工作内存中                     |
| load   | 将从主内存中通过read操作获得的变量值，放入工作内存的副本中   |
| use    | 将工作内存中一个变量值传递给执行引擎，进行使用（使用变量时调用） |
| assign | 将执行引擎接收到的值，赋值给工作内存中变量（赋值操作时调用） |
| store  | 将工作内存中的一个变量传递到主内存中                         |
| write  | 将从工作内存中通过store操作获得的变量值，写入到主内存中      |

#### 2.3 介绍下volatile关键字？

**作用**：

- 当线程读取volatile修饰的变量，会被要求从主内存中重新读取，当写一个volatile修饰变量时，会被要求刷新至主内存中，从而保证了可见性

- 通过内存屏障，禁止了指令重排序，保证了有序性
- 不能保证原子性

**底层实现**：

- 对volatile修饰的变量进行写操作时，底层会添加个Lock前缀，
- Lock前缀会强制将工作内存中变量写回到主内存
- 此时为了避免其它线程去从主内存中读取该变量值，会对缓存该变量的区域加锁，使得在变量修改过程中（变量从工作内存写入主内存过程中），其它的线程不能够去主内存中读取该变量，当更新完成后，unlock后才能读取
- 同时，其它线程是如何知道该变量已被其他线程修改，需要再从主内存中读取？依据缓存一致性协议，CPU不断嗅探总线上的数据交换，能够得知数据的变化，当volatile修饰的变量发生变化时，对应缓存中存储的变量会变为失效状态，需要从主内存中再读取一遍，从而保证了变量为最新值

![volatile流程](.\img\03\03_03.PNG)



#### 2.4 volatile为什么能够保证可见性和有序性？

- 可见性，当线程读取volatile修饰的变量，会被要求从主内存中重新读取，当写一个volatile修饰变量时，会被要求刷新至主内存中，从而保证了可见性
- 有序性，volatile会插入lock指令，具备内存屏障，内存屏障会保证在屏障之前的必须先执行，之后的后执行，禁止了重排序，从而保证了有序性
- 不能保证原子性

#### 2.5 volatile为什么不能保证原子性？

以多线程进行i++为例，本身i++不具备原子性。

同时在底层中，当线程2执行i++操作时，从主内存中读取i进行加1，放到工作内存中，此时另外一个线程获得CPU使用权，也从主内存读取i值，进行i++操作，执行完毕后，写入到主内存。此时，线程2已完成+1操作，还未写入主内存，但被告知主内存变量已被修改，导致完成+1操作的变量，存放在工作内存失效，此次+1不成功。

**总结**：即使volatile修饰的变量，也不能保证底层工作内存与主内存数据交互指令的原子性，可能会导致已经写入工作内存的变量失效，使得少了一次+1操作，因此实际值会比期望值小或者相等。

![volatile不具备原子性](.\img\03\03_04.PNG)

#### 2.6 什么是先行并发原则（happends-before）?







## 三 Java对象模型

#### 3.1 对象的内存结构？

对象的内存结构分为3个部分，对象头（Header）、实例数据（Instance Data）、对齐填充（Padding）。

- 对象头，包含两部分信息，为Mark World和元数据指针。Mark World用于存储对象运行时数据，如HashCode、锁状态标志、偏向锁线程ID、GC分代年龄、数组对象长度（为数组对象才有）等。元数据指针用于指向方法区（元数据区）中的类的类型信息，通过元数据指针能确定对象的具体类型。
- 对象实例数据，创建对象时，对象中的成员变量方法等。
- 对齐填充，对齐填充不一定存在，起到了占位符作用。



Java对象基于OOP-Klass二分模型，HotSpot虚拟机会将Java对象利用C++在虚拟机中一一对应。

OOP指普通对象指针，Klass是C++中对等于Java中的对象。

#### 3.2 实例对象是怎么存储的？

对象的实例存储在堆空间，对象的元数据存储在方法区（元空间区），对象的引用存储在栈空间







## 四  线程

#### 4.1 什么是用户线程和内核线程？

- 用户线程  

  创建、同步、销毁和调度完全在用户态中完成，不需要内核参与。一个进程中，可以有多个用户线程。

- 内核线程

  直接由操作系统内核支持的线程，由系统内核来完成线程切换，内核通过操作调度器对线程进行调度，并负责将线程的任务映射到各个处理器上。一般一个CPU核心对应一个内核线程，比如单核处理器对应一个内核线程，双核处理器对应两个内核线程，四核处理器对应四个内核线程

#### 4.2 Java中的线程？

​	Java中的线程创建依赖于系统内核，创建线程时JVM会调用系统库去创建内核线程，一条Java线程会映射到一条系统的轻量级进程上（指通常说的线程），1对1。



#### 4.3 线程与进程？





#### 4.4 线程的创建？

四种方式

- 继承Thread 类

Thread类实现了Runnable接口，该接口中声明了抽象方法run()。Thread类代表一个线程实例，通过调用start()方法启动线程，将会回调run()方法，该方法称为线程体，重写后实现自己的任务。

```java
class MyThread extends Thread {
	@Override
	public void run() {
		System.out.println("run run run");
	}
}
MyThread myThread = new MyThread();
myThread.start();
```

- 实现Runnable接口

Runnable为一个接口，先声明一个类实现该接口，主要用于重写run()方法。然后实例化该类，作为参数传递到Thread(Runnable target)类中。初始化线程时，会赋值给线程类的target属性。

```java
class MyRunnable implements Runnable {
	@Override
	public void run() {
		System.out.println("run run run");
	}
}
Thread myThread = new Thread(new MyRunnable());
myThread.start();
```

能够通过该种方式创建线程，是因为run()方法默认实现如下。回调run()方法时，会先判断target是否为null，target会在执行new Thread(new MyRunnable())时，在构造方法中赋值。

```java
private Runnable target;
public void run() {
    if (target != null) {
        target.run();
    }
}
```

- 使用Callable、FutureTask方式

使用实现接口Callable<V>，该种方式通过重写call()方法，实现自己的任务。同时能够调用FutureTask对象的get()方法来获得子线程执行结束后的返回值。FutureTask<V>也实现了Runnable接口，然后重写了run()方法，在该方法中会去调用重写的call()方法，并且获取返回值。

```java
class MyCallable implements Callable<Integer> {
	@Override
	public Integer call() throws Exception {
		System.out.println("run run run");
		return 111;
	}
}
FutureTask<Integer> ft = new FutureTask<>(new MyCallable());  
Thread myThread = new Thread(ft, "hasRetThread");
myThread.start();
```

- 创建线程池实现

使用线程池创建并管理线程，能够通过重复利用已创建的线程来降低资源消耗，可以使用线程池统一分配、监控线程。newCachedThreadPool()为线程池一种，还可以根据需求使用其它方法。

```java
ExecutorService executorService = Executors.newCachedThreadPool();
executorService.execute(new Runnable() {
    @Override
    public void run() {
    	System.out.println("run run run");
    }
});
```



#### 4.5  Runnable与Callable的区别？

Callable的设计在于既能够实现像Runnable一样创建任务，同时还能够获取任务执行完后的返回值，能够监控任务执行

- Callable规定的方法是call()，而Runnable规定的方法是run()。
- Callable的任务执行后可返回值，而Runnable的任务是不能返回值的。
- call()方法可抛出异常，而run()方法是不能抛出异常的。
- 运行Callable任务可拿到一个Future对象，通过future.get()能够获取返回值，会阻塞

#### 4.6 线程的生命周期？



####  4.7 启动线程为什么不能直接调用run方法，而是要通过start？



## 五 线程池

#### 5.1 为什么要用线程池？线程池的优势？

阿里巴巴手册建议

**强制：线程资源必须通过线程池提供，不允许应用自行显式创建线程**

线程池好处，能够减少在创建和销毁线程上所消耗的时间及资源的开销，。

如果不使用线程池，可能造成系统创建大量同类线程而导致消耗完内存或者过度切换问题。



#### 5.2 3种常见线程池？

Executors为java.util.concurrent下的工具类，提供工厂方法来创建不同的线程池。常用方法如下：

- newFixedThreadPool(int nThreads)     

传入一个固定线程数大小，corePoolSize=maximumPoolSize=nThreads。任务队列为LinkedBlockingQueue，默认容量为Integer.MAX_VALUE。

线程数固定，可执行长期任务

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
```

- newCachedThreadPool()

可缓存线程池，实现代码如下。corePoolSize为0，maximumPoolSize为Integer.MAX_VALUE。当调用execute方法时，会重用先前构造的线程，如果没有可用的线程，将创建一个新线程并将其加入到池中。未使用超过60秒的线程将终止并从缓存中删除。 因此，长时间空闲的池不会消耗任何资源。

适用很多短期异步小任务或负载较轻的服务器。

线程来了就能执行，因此阻塞队列采用SynchronousQueue，只能存放一个元素。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

- newSingleThreadExecutor()

创建一个只有一个工作线程的线程池。corePoolSize为1，maximumPoolSize为1.

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
```



- newScheduledThreadPool(int corePoolSize)

创建一个能够延时启动，并定时周期性循环的线程池。



#### 5.3 线程池的几个重要参数？

- corePoolSize：  核心线程数量，即使空闲，也不会终止（类似于今日某银行网点的值班窗口）
- maximumPoolSize： 最大线程数，线程池里能够容纳同时执行的最大线程数量，值必须大于1。（银行网点最多有这么个窗口，大于corePoolSize的可理解为今日可加班的窗口）
- workQueue： 任务队列，用于保存等待执行任务的阻塞队列，只包含有execute方法提交的Runnable。（类似于银行网点的候客区）
- keepAliveTime：空闲线程最大等待时间，当存在线程数大于核心线程数量，多余线程会在等待keepAliveTime长后销毁，直至线程数等于corePoolSize。（当人数变少了，加班窗口等待keepAliveTime时间后，没有客户，可以关闭）
- threadFactory：设置生成线程池中工作线程的线程工厂，可通过工厂为线程设置独特名称，一般默认。
- handler：线程池拒绝策略，当队列满了且工作线程数大于等于线程池中的最大线程数（maximumPoolSize），对新提交任务的处理策略。默认为AbortPolicy，抛出RejectedExecutionException异常。（银行窗口都满了，候客区也满了，其余再来人员就要拒绝）

流程，当corePoolSize满了，先去阻塞队列排队，阻塞队列满了，还能去启动（maximumPoolSize-corePoolSize）个线程，当都满了，再执行拒绝策略。

可通过联想银行网点、检票窗口等实际场景理解。



#### 5.4 线程池的工作流程？

流程，当任务来了之后，先提交给corePool，当corePoolSize满了，先去阻塞队列排队，阻塞队列满了，还能去启动（maximumPoolSize-corePoolSize）个线程，当都满了，再执行拒绝策略。

当执行一段时间后，需要执行的线程数变少，（maximumPoolSize-corePoolSize）个线程，会在等待keepAliveTime后关闭。corePool中的线程永远不会释放。

![线程池执行流程](E:\tt文档\我的文档\ttGit\JavaInterview\img\03\03_05.PNG)

1. 创建了线程池后，线程池会等待任务提交

2. 调用execute()方法提交任务后，线程池会进行如下处理

   1）如果正在运行的线程数小于corePoolSize，将马上创建线程并运行

   2）如果正在运行的线程数大于或等于corePoolSize，将会去加入到阻塞队列中

   3）当阻塞队列满了，且正在运行的线程数小于maximumPoolSize，会去创建非核心线程数并立刻去执行

   4）当阻塞队列满了，且正在运行的线程数量等于maximumPoolSize时，线程池会启动拒绝策略，默认为AbortPolicy，抛出RejectedExecutionException异常。

1. 当一个线程执行完，会从队列中取下一个任务来执行

1. 当一个线程没有执行任务，至超过一定时间keepAliveTime后，且当前运行的线程数大于corePoolSize，那么该线程会被停掉，直到线程数等于corePoolSize。

#### 5.5 如何配置线程池拒绝策略？

当等待队列满了，正在运行的线程数等于maximumPoolSize时，对于新加入线程，线程池会执行拒绝策略。

**包含哪些拒绝策略**

- AbortPolic(默认)，直接抛出RejectedExecutionException异常
- CallerRunsPolicy，既不抛弃任务，也不抛出异常，而是将某些任务回退到调用这，从而降低新任务排队
- DiscardOldestPolicy，直接抛弃等待最久的任务，然后把当前任务加入队列中尝试再次提交执行
- DiscardPolicy，直接丢弃任务，不予任何处理也不抛出异常。



#### 5.6 线程池如何创建？为什么不使用 Executors 去创建？

在阿里巴巴Java开发手册中，明确列出

> 【强制】线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样
> 的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。
> 说明： Executors 返回的线程池对象的弊端如下：
> 1） FixedThreadPool 和 SingleThreadPool:
> 允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。
> 2） CachedThreadPool 和 ScheduledThreadPool:
> 允许的创建线程数量为 Integer.MAX_VALUE， 可能会创建大量的线程，从而导致 OOM。 

因此，推荐直接调用`ThreadPoolExecutor`，手动设置符合应用场景的参数，或者使用guava提供的`ThreadFactoryBuilder`来创建线程池。

FixedThreadPool 和 SingleThreadPool使用的阻塞队列为LinkedBlockingQueue

CachedThreadPool 为SynchronousQueue



#### 5.7 如何合理配置线程池参数？

并发编程中，使用多线程，是为了提升性能，即提高硬件的利用率，就是提升 I/O 的利用率和 CPU 的利用率。并发中执行的任务主要为CPU密集型任务和IO密集型任务。

- 对于CPU密集型任务，单个线程CPU使用率高，如果设置线程数过大，线程之间不断切换，会带来额外开销。因此，可根据CPU核心数，设置可能小的线程数，如设置（CPU核心数+1）个线程，加1是为了避免某个线程由于特殊原因造成阻塞，可用于切换。
- 对于IO密集型任务，并不需要一直执行任务，可多配置线程数。理论上，通过计算任务占用CPU时长与占用IO时长，若为1:1，则设置两个线程，可以100%利用CPU和IO资源。因此，可以采用CPU 核数 * [ 1 +（IO耗时 / CPU 耗时）]，估算参考值。

对于密集型任务，也有建议使用（2*CPU核心数）估算，为什么CPU系数选择2，实际为默认IO耗时 ：CPU 耗时 = 1 ：1，显然，如果能够得到（IO耗时 / CPU 耗时），则估算会更为准确。

然而，在实际应用时，上述线程数设置值并一定就准确，只是一个估计值，需要进行测试，关注性能参数，并进行调整才更为合理，才能最有效降低响应耗时和增大吞吐量。



#### 5.8 线程池底层代码实现？





## 六 AQS

#### 6.1 AQS的实现原理？核心为3点

***第一步***

**主要思想**，多个线程去请求共享资源，为保证有序，若资源为锁定状态，AQS通过线程阻塞，等待被唤醒，合理有效分配共享资源使用，保证线程安全。AQS中即实现了这一机制。

通过以下三种保证多个线程去竞争共享资源只有一个线程能够成功。

**核心三点**

- CAS、保证共享状态state的修改，保证原子性
- 自旋、控制线程在自己范围内，
- LockSupport的park/unpark，避免线程一直自旋空跑，通过park/unpark使线程根据需要阻塞和唤醒

**采用了模板方法模式**，AQS为抽象类，使用者需要重新AQS中的指定方法，主要是对同步状态state获取和释放，来适配锁的模式。而AQS中的模板方法会去调用使用者重写的那些方法。然后使用者再调用那些模板方法来构建锁。

***第二步***  存放阻塞线程的队列

AQS用于存放管理阻塞线程的队列，该队列是基于CLH变种的队列，是一种基于双向链表的队列。该队列只存在节点与节点之间的关联关系，不向线程池那样存在实例化的队列。

队列中存放的节点代表一个线程节点Node，其中包含线程、用volatile修饰线程的等待状态信息waitStatus、前继后继节点。该状态包含5个状态值，后面主要是通过CAS，来修改每个节点的状态信息，不同状态信息表示节点处于同步队列还是等待队列，是否被取消，是否状态被唤醒等。队列中排队的线程主要通过判断前继节点的状态来决定是否能够执行。

![同步队列与等待队列](.\img\03\03_08.PNG)



***第三步***  线程节点加入/离开同步队列大致流程

线程在尝试获取锁时，能够获取到同步状态，则直接执行，此步是通过CAS设置state值，期望值为0，表示没有线程占有，设定值为1，表示该线程想要去获取到同步状态，如果CAS失败，则表示未获取到锁，需要去排队。

若获取不到，需要插入到该同步队列尾部。通过自旋，CAS方式进行插入，保证原子性。插入进等待队列后，会通过LockSupport.park(this）方法进入阻塞状态，等待被唤醒。

非公平情况，新来一个线程会先去直接去竞争锁，如未竞争到，再去排队。竞争到锁，直接能够执行。

而公平锁则是直接去排队。

共享情况下，当当前节点被唤醒后，会去尝试唤醒后面节点，进行传播。

***第四步***  等待队列过程

在独占锁中才能使用等待队列，共享模式不能。

等待队列指队列中每个排队线程都在等待对应的条件，不同等待队列的条件不一样。当线程获取到锁，调用await()方法将从同步队列转移到等待队列中，此时不需要通过CAS保证线程安全，因为是先获得了锁，才能执行await()方法，线程进入对应条件的阻塞，然后释放锁。直到有线程竞争到锁，调用对应条件的signal()方法才能将该等待队列中的头节点，移动到同步队列中去竞争锁。

**第五步** 应用

在ReentrantLock（可重入锁）中，需要先创建内部类sync，去继承AQS，重写指定方法。初始状态state为0，线程A调用lock()方法时，会调用AQS中的tryAcquire()方法，来竞争获得锁，获得锁后，state+1，其它线程不能会被加入同步队列阻塞，当线程A再去获取锁，能够直接获得，即可重入，此时state会再原先基础上再+1，同时线程A在释放锁时，也会将state减1，直到state为0，才完全释放锁。

在CountDownLatch中，为共享模式，初始化时设置了需要等待执行完成的线程个数Count，会将该值赋值给AQS中的state。当调用线程调用await()方法阻塞等待所有线程完成时，当某个线程执行countDown方法，state值会减1，直到state为0，主线程会被唤醒从await()方法返回，从而能够继续执行。

在阻塞队列ArrayBlockingQueue中，即通过该种方式实现。在阻塞队列中通过notFull和notEmpty两个条件

。当元素插入阻塞队列时，队列满了，元素将进入等待条件为notFull的等待队列，等待被唤醒；当有线程去获取元素时，取出元素后，会再调用notFull.signal()方法，去唤醒等待条件为notFull的等待队列中元素，因此元素又可以被插入了。notEmpty同理。



#### 6.2 await()与signal()的等待通知机制

Java中可以采用`wait()/notify()`方法实现等待通知机制，同样也可以采用`await()/signal()`方法实现。`await()/signal()`是AQS中提供的方法。

线程在获取到锁lock后，可通过调用condition.await()方法进入等待队列，被阻塞，并释放锁lock。其它线程竞争到锁，执行完操作后，可以调用condition.await()方法也进入等待队列。每一个等待condition都对应一个等待队列，AQS中同步队列只有一个，等待队列可以有多个。当某个线程在获取到锁lock后，调用condition.signal()方法，会将等待队列中首节点，移动到同步队列。此时，同步队列中元素可以去竞争锁。



## 七 Java中的锁

#### 7.1 何为公平锁和非公平锁？

**公平锁**

多个线程按照申请锁的顺序来获取锁，依次排队

**非公平锁**

非公平锁，多个线程获取锁不是按照申请锁的顺序，可能后申请锁的线程能够比先申请锁的线程先拿到锁，

**区别**

公平锁，并发环境下，线程在申请获得锁时，会查看此锁维护的同步队列是否为空，或者该线程排在第一位，那么将会直接获得锁，否则会直接加入等待该锁的同步队列尾部，优先按照同步队列顺序FIFO获得锁。将会导致每个线程都需要被再唤醒。

非公平锁，线程在申请锁时，直接尝试竞争占有锁，如果尝试失败，则按照公平锁操作来获得锁。因为不是每个线程都需要排队，可以减少唤醒同步队列中等待线程的次数，但是处于同步队列中的线程可能会多次都竞争不到锁，造成饥饿。

**应用**

并发包中的ReentrantLock默认为非公平锁

Synchronize为非公平锁

#### 7.2 何为可重入锁？

**可重入锁**为线程可以在外部方法获得锁后，内部代码仍然需要获取该同一个锁，能够自动获取，不会因为外部方法未释放锁，导致阻塞，陷入死锁。

**作用**：能够有效避免死锁。

Java中ReentrantLock 和synchronized都是可重入锁。

**实例**

```java
    public synchronized void doSth() {
        System.out.println(Thread.currentThread().getId());
        // 能够成功执行该方法
        doOtherSth();
    }

    public synchronized void doOtherSth() {
        System.out.println(Thread.currentThread().getId());
    }
```

#### 7.3 何为自旋锁？何为自适应自旋锁？

**是什么**

一个线程获取锁时，锁被其它线程占用，该线程不断尝试去获取锁，即自旋。自旋锁并非真正意义上的锁，Java没有提供相应API可以直接调用，是一种锁优化技术。

**好处**    不会阻塞，减少了线程的上下文切换，适用于锁竞争情况少

**缺点**    自旋次数过多，会占用CPU资源

**示例**

```java
public class SpinLock
{
    //Java中的原子操作（CAS）
    //持有自旋锁的线程对象
    AtomicReference<Thread> owner = new AtomicReference<Thread>();

    public void lock()
    {
        Thread curThread = Thread.currentThread();
        //lock函数将owner设置为当前线程，并且预测原来的值为空
        //当有第二个线程调用lock操作时由于owner的值不为空，导致循环
        //一直被执行，直至第一个线程调用unclock函数将owner设置为null，第二个线程才能进入临界区
        while(!owner.compareAndSet(null, curThread))
        {
        }
    }

    //unlock将owner的值设置为null，并且预测值为当前线程
    public void unlock()
    {
        Thread cur=Thread.currentThread();
        owner.compareAndSet(cur, null);
    }

}
```

#### 7.4 独占锁（写锁）/共享锁（读锁）/互斥锁

**独占锁**  该锁一次只能被一个线程锁持有。 ReentrantLock和Synchronized都是独占锁

**共享锁**  锁可以被多个线程持有。ReentrantReadWriteLock读锁为共享锁，写锁时独占锁



#### 7.5 为什么要有锁？

硬件性能的提升，需要并发、并行，但是还是需要保证线程安全。

锁用来保证多个线程访问修改共享资源时，能够得到正确地结果，即线程安全。

锁主要包含内置锁synchronized和显示锁ReentrantLock等。

其中，AQS是Java中多种锁、并发工具类实现的基础。



## 八 并发工具类

#### 8.1 CountDownLatch

**定义**，一个或多个线程需要等待其它线程执行完，才能继续执行。

**用法**，

当一个线程或多个线程调用CountDownLatch的await方法，调用线程会被阻塞。其它线程调用CountDownLatch的countDown方法计数器会减1，当初始计数器值减为0时，调用await方法被阻塞的线程会被唤醒，继续执行。

**应用**，

可以模拟运动员一起起跑；

所有运动员都到达终点后，宣布结束；

**示例**

```java
    public static void main(String[] args) {

        CountDownLatch countDownLatch = new CountDownLatch(5);
        for (int i = 0; i < 5; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {                    System.out.println(Thread.currentThread().getName());
                    countDownLatch.countDown();
                }
            }, "thread " + i).start();
        }

        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("all threads finish.");
    }
```

#### 8.2 CyclicBarrier 栅栏

**定义**，可循环的屏障。让一组线程都达到一个屏障（也可称同步点）时被阻塞，直到最好一个线程到达屏障时，屏障会被打开，所有线程才能继续执行，线程进入屏障通过await()方法。

构造方法时能够传入barrierAction，用于在线程到达屏障时，优先执行barrierAction，然后被屏障阻塞线程再执行。

**示例**

```java
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> System.out.println("开始跑。。。"));
        for (int i = 0; i < 5; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {

                    try {
                        cyclicBarrier.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }

                    System.out.println(Thread.currentThread().getName());

                }
            }, "thread " + i).start();
        }
    }
```

#### 8.3 Semaphore 信号量

**定义**：1. 用于多个共享资源的互斥，2. 控制并发线程数

Semaphore 通过acquire()能够获取一个许可，release()释放一个许可。如果同时访问的任务数等于设定的许可总数，那么尝试获取许可的任务将进入阻塞状态，直到有一个任务执行完释放了许可后，其它任务才能获取到许可。

能够用于流量控制。

**示例**

如以下代码，通过信号量限制，将每间隔1s，打印一次当前线程名称。

```java
	public static void testSemaphore() {
		int threadCount = 10;
		Semaphore semaphore = new Semaphore(1);
		ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
		
		for (int i = 0; i < threadCount; i++) {
			executorService.execute(new Runnable() {
				@Override
				public void run() {
					try {
						semaphore.acquire();
						
						System.out.println(Thread.currentThread().getName());
						Thread.sleep(1000);
						
						semaphore.release();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
			});
		}
	}
```

#### 8.4  CountDownLatch和CyclicBarrier 对比

- CountDownLatch的计数器值不能重用，一次性使用，当执行完成后，需要重新初始化。而`CyclicBarrier`能够重置，当屏障打开后，所有线程操作执行完成后，会再关闭，开启下一代。
- CountDownLatch，一个线程或多个线程等待其它N个线程达到某个条件后，才能继续执行。而CyclicBarrier是要等待多个线程是否到达一个同步点，才能继续执行。
- CountDownLatch是await()和countDown()方法的组合使用，需要等待的线程通过await()方法阻塞，其它线程通过countDown()方法将计数器减1。CyclicBarrier主要是使用await()方法，当所有的线程都调用了await()方法，才能继续往下执行。





## 九 阻塞队列

#### 9.1 什么是阻塞队列

- 当队列中没有元素时，从队列中获取元素，会被阻塞
- 当队列已满，向队列中插入元素，会被阻塞

从空的阻塞队列中获取元素的线程会被阻塞，直到其它线程往空的阻塞队列中插入元素。

向满的阻塞队列中插入元素的线程会被阻塞，直到其它线程从阻塞队列中移除一个或多个元素

#### 9.2 为什么要有阻塞队列？阻塞队列有什么好处？如何管理正在阻塞的元素？

阻塞时，线程会被挂起，条件满足，挂起线程又会被唤醒。

为什么需要？有了阻塞队列，不需要认为去控制线程的阻塞和挂起，比较方便。

#### 9.3 常用阻塞队列

`Jdk8`中常见以下阻塞队列。

- **ArrayBlockingQueue**，一个基于数组结构实现的有界阻塞队列。初始化时需指定数组容量 。插入和移除元素时，共用一把锁。（重点）
- **LinkedBlockingQueue**，一个基于链表结构实现的有界阻塞队列。初始化时指定容量，不设置默认为`Integer.MAX_VALUE` （约21亿）。插入和移除元素，使用两个锁对象。（重点）
- **DelayQueue**，一个支持优先级排序和延时获取元素的无界阻塞队列。插入元素不会被阻塞，队里中元素只有当指定的延迟时间到了，才能够从队列中获取该元素，否则会阻塞。
- **PriorityBlockingQueue**，一个支持优先级排序的无界阻塞队列。
- **SynchronousQueue**，只能存储一个元素，当存在一个元素时，再插入元素会阻塞，直到元素被移除，才能正常插入。（重点）

#### 9.4 阻塞队列的核心方法？

| 方法类型 | 抛出异常  | 特殊值   | 阻塞   | 超时                 |
| -------- | --------- | -------- | ------ | -------------------- |
| 插入     | add(e)    | offer(e) | put(e) | offer(e, time, unit) |
| 删除     | remove()  | poll()   | take() | poll(time, unit)     |
| 检查     | element() | peek()   | 不支持 | 不支持               |

- 抛出异常

当队列满时，再往队列里add元素，会抛IllegalStateException: Queue full异常

当队列为空时，再去remove元素，会抛java.util.NoSuchElementException异常

- 特殊值

插入方法，成功ture，失败false

移除方法，成功返回出队列的元素，队列里没有返回null

检测方法，队列没有元素，返回null，存在返回第一个元素

- 阻塞

插入元素，满了阻塞，直到队列中有元素

移除方法，空了阻塞

#### 9.5 SynchronousQueue队列说明

只能存储一个元素，当存在一个元素时，再插入元素会阻塞，直到元素被移除，才能正常插入。

**示例**

```java
public static void testSynchronousQueue() {
    SynchronousQueue<String> stringSynchronousQueue = new SynchronousQueue<>();

    new Thread(() -> {
        try {
            System.out.println(Thread.currentThread().getName() + " put aaa");
            stringSynchronousQueue.put("aaa");
            System.out.println(Thread.currentThread().getName() + " put bbb");
            stringSynchronousQueue.put("bbb");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }, "AAA").start();

    new Thread(() -> {
        try {
            TimeUnit.SECONDS.sleep(5);
            System.out.println(Thread.currentThread().getName() + " take aaa");
            System.out.println(stringSynchronousQueue.take());

            TimeUnit.SECONDS.sleep(5);
            System.out.println(Thread.currentThread().getName() + " take bbb");
            System.out.println(stringSynchronousQueue.take());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }, "BBB").start();
}
```



#### 9.6 阻塞队列应用

**消费者生产者模式**（await/signalAll方式）

```java
/**
 * 题目：一个初始值为0的变量，两个线程不断进行+1  -，进行5轮
 */
public class TestProConsumer {
    public static void main(String[] args) {
        ProAndCon proAndCon = new ProAndCon();
        for (int i = 0; i < 5; i++) {
            proAndCon.increase();
        }

        for (int i = 0; i < 5; i++) {
            proAndCon.decrease();
        }

        for (int i = 0; i < 5; i++) {
            proAndCon.increase();
        }

        for (int i = 0; i < 5; i++) {
            proAndCon.decrease();
        }
    }
}
class ProAndCon {
    private int value = 0;
    final Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    /**
     * 生产
     */
    public void increase() {
        new Thread(() -> {
            lock.lock();

            try {
                // 此处只能用while判断，if多线程判断会出错
                while(value != 0) {
                    condition.await();
                }
                value++;
                System.out.println(Thread.currentThread().getName() + " " + value);
                condition.signalAll();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }, "AAA").start();
    }

    /**
     * 消费
     */
    public void decrease() {
        new Thread(() -> {
            lock.lock();
            try {
                while(value == 0) {
                    condition.await();
                }
                value--;
                System.out.println(Thread.currentThread().getName() + " " + value);
                condition.signalAll();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }, "BBB").start();
    }
}
```

**消费者生产者模式**（）

```java

```



## 十 synchronized关键字

#### 10.1 synchronized用法

主要有三种用法，一是修饰实例方法，二是修饰静态方法、三是修饰代码代码块。

修饰实例方法，锁对象是当前对象实例。

修饰静态方法，锁对象是当前类。

修饰代码块，锁对象是指定传入对象。

```java
public class TestSynchronized {
	public synchronized void test1() {
		System.out.println("test synchronized");
	}
	
	public void test2() {
		synchronized (TestSynchronized.class) {
			System.out.println("test synchronized");
		}
	}
}
```

被synchronized修饰的方法或者代码块，同时只能有一个线程执行，由于synchronized为内置锁，只查看Java代码，不能看出两者区别，通过反编译class文件，可以发现不同。

```java
  public synchronized void test();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #15                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #21                 // String test synchronized
         5: invokevirtual #23                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return

  public void test2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=1
         0: ldc           #1                 
         2: dup
         3: astore_1
         4: monitorenter
         5: getstatic     #15                
         8: ldc           #21  // String test synchronized
        10: invokevirtual #23  // Method java/io/PrintStream.println(Ljava/lang/String;)V
        13: aload_1
        14: monitorexit
        15: goto          21
        18: aload_1
        19: monitorexit
        20: athrow
        21: return
     Exception table:
         from    to  target type
             5    15    18   any
            18    20    18   any
```

#### 10.2 synchronized的实现？

synchronized是内置锁，可重入，内部通过Monitor监视器实现。

synchronized修饰方法，flags多了`ACC_SYNCHRONIZED`字段，用于表明该方法被关键字synchronized修饰，为同步方法。线程在执行到方法时，发现有`ACC_SYNCHRONIZED`标志，会先去获取监视器，获取到监视器，继续执行，执行完毕后释放监视器。后来的线程，获取不到监视器，会被阻塞。

synchronized修饰代码块，同步代码块，前后多了`monitorenter`和`monitorexit`指令，用于表明同步代码块的开始与结束。线程执行到`monitorenter`后，会去尝试获取锁对象monitor，该对象存放在Java对象的对象头中，不同的配置表示不同的锁类型，每个Java对象存在对象头，因此都可以作为锁对象。当计数器为0，就可以成功获取，获取后计数器值加1。在执行到`monitorexit`时，计数器值减1，为0时，即释放锁。如果获取对象锁失败，会被阻塞，直至获取到锁。

如上反编译后，同时多了异常表，用于捕获同步代码块中异常，当5-15行出现 异常，跳转第18行执行，再次执行一次`monitorexit`指令，用于释放锁。如果没有异常，则会正常执行第14行进行释放锁。因此，synchronized也是会通过try-finally来隐式释放锁。

**以下为扩展**

解释器在执行`monitorenter`指令时，会先通过解释器，包含模板解释器和字节码解释器，默认会使用模板解释器，位于[templateInterpreter.cpp](http://hg.openjdk.java.net/jdk/jdk/file/6659a8f57d78/src/hotspot/share/interpreter/templateInterpreter.cpp)，其中字节码对应机器码模板位于[templateTable.cpp](http://hg.openjdk.java.net/jdk/jdk/file/6659a8f57d78/src/hotspot/share/interpreter/templateTable.cpp)中。针对不同平台，`monitorenter`与`monitorexit`的实现不同，X86平台对应代码[templateTable_x86_64.cpp](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/cpu/x86/vm/templateTable_x86_64.cpp#l3667)。模板解释器是在运行时会依据字节码模板表、抽象解释器生成本地代码，之后对指令解析是通过该代码。因此，也可以通过查看字节码解释器，查看`monitorenter`处理逻辑，对应代码[bytecodeInterpreter.cpp#1816](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/bytecodeInterpreter.cpp#l1816) 。在[文章](https://github.com/farmerjohngit/myblog/issues/13)中，该作者对该代码解释十分清楚，可参考。最终都会解析进入相应的指令方法中，位于[interpreterRuntime.cpp](http://hg.openjdk.java.net/jdk/jdk/file/6659a8f57d78/src/hotspot/share/interpreter/interpreterRuntime.cpp)的InterpreterRuntime::monitorenter和InterpreterRuntime::monitorexit方法中。

InterpreterRuntime::monitorenter方法代码如下。

```c++
//%note monitor_1
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
  if (PrintBiasedLockingStatistics) {
    Atomic::inc(BiasedLocking::slow_path_entry_count_addr());
  }
  Handle h_obj(thread, elem->obj());
  assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
         "must be NULL or an object");
  if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
  assert(Universe::heap()->is_in_reserved_or_null(elem->obj()),
         "must be NULL or an object");
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
IRT_END
```

1. 方法入参

   - 参数thread，表示当前线程；

   - BasicObjectLock类型的elem变量，对象内部对象由一个BasicLock对象`_lock`和一个指向持有该锁的Java对象指针`_obj`组成，`_obj`用于指向对象头数据。

     ```c++
     class BasicObjectLock {
       BasicLock _lock; 
       // object holds the lock;
       oop  _obj;   
     }
     ```

   - 其中，BasicLock对象中包含`markOop _displaced_header;`变量，用于表示对象头数据。`markOop `用于描述对象Mark Word。

     ```c++
     class BasicLock VALUE_OBJ_CLASS_SPEC {
       	...
          private:
           volatile markOop _displaced_header;
          public:
           markOop  displaced_header() const   { return _displaced_header; }
           void   set_displaced_header(markOop header)  { _displaced_header = header; }
     	...
     };
     ```

2. JVM中使用-XX:+UseBiasedLocking， 设置启用偏向锁。此时会走fast_enter()分支。



#### 10.3 锁的膨胀升级？

锁的升级依次为，偏向锁->轻量级锁->重量级锁。

| 锁类型   | 定义                                                       | 应用场景                                                     |
| -------- | :--------------------------------------------------------- | ------------------------------------------------------------ |
| 无锁     | 没有锁，普通对象                                           |                                                              |
| 偏向锁   | 同一个线程申请锁时，能够直接获取                           | 只有一个线程进入临界区，没有竞争，适用于只有一个线程访问同步代码块。在锁竞争激烈的场合没有太强的优化效果。 |
| 轻量级锁 | 多个线程竞争同步资源时，没有获取到锁的线程，自旋等待锁释放 | 多线程未竞争或竞争不激烈，适用于同步代码块执行速度快，追求响应时间 |
| 重量级锁 | 多个线程竞争同步资源时，没有获取到锁的线程，阻塞等待唤醒   | 多线程存在竞争，适用于同                                     |

对象头中锁状态信息也会随着锁的升级进行变化

| 锁状态   | 锁标志位 | 包含内容                                                    |
| -------- | -------- | ----------------------------------------------------------- |
| 无锁     | 01       | 对象hashcode、对象分带年龄                                  |
| 偏向锁   | 01       | 偏向线程的ID、（Epoch）偏向时间戳、对象分代年龄、是否可偏向 |
| 轻量级锁 | 00       | 指向当前占有锁的线程的栈中锁记录指针（lock record）         |
| 重量级锁 | 10       | 指向锁对象的指针                                            |



#### 10.4 Synchronized和lock有什么区别？用新的lock有什么好处？

**区别**

- 原始构成上，实现原理上

Synchronized是关键字，属于JVM层面

​	monitorenter(底层通过monitor对象完成，wait和notify也依赖于monitor实现，需要在同步代码块中调用)

​	monitorexit

Lock是API层面的锁

- 使用方法上

synchronized不需要用户手动去释放锁，代码执行完，系统会自动去释放锁。

Lock需要用户去手动释放锁，通过配合try/finally来完成

- 等待是否可中断（lock的好处）

Synchronized不可中断，除非抛出异常或者运行完成。

ReentrantLock可中断，通过设置超时方法tryLock(Long timeout, TimeUnit unit)；通过interrupt()方法中断

- 加锁是否公平

synchronized非公平锁

ReentrantLock,默认非公平锁，构造方法传入true为公平锁，false为非公平锁

- 锁绑定多个条件Condition（lock的好处）

Synchronized不支持

ReentrantLock可以实现分组唤醒需要唤醒的线程，实现精确唤醒

**示例** (多条件Condition应用)

```java
/**
 *  多线程之间按顺序调用，A-B-C三个线程
 *  AA打印5次，BB打印10次，CC打印15次
 *
 *  循环5轮
 */
public class TestSyncAndLock {
    public static void main(String[] args) {
        CyclicPrint cyclicPrint = new CyclicPrint();

        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                cyclicPrint.print5();
            }, "AA").start();
        }

        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                cyclicPrint.print10();
            }, "BB").start();
        }

        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                cyclicPrint.print15();
            }, "CC").start();
        }
    }
}

class CyclicPrint {

    private int number = 1; //A 1， B 2，C 3

    private ReentrantLock reentrantLock = new ReentrantLock();
    private Condition c1 = reentrantLock.newCondition();
    private Condition c2 = reentrantLock.newCondition();
    private Condition c3 = reentrantLock.newCondition();

    public void print5() {
        reentrantLock.lock();
        try {
            while (number != 1) {
                c1.await();
            }
            for (int i = 0; i < 5; i++) {
                System.out.println("AA");
            }
            number = 2;
            c2.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            reentrantLock.unlock();
        }
    }

    public void print10() {
        reentrantLock.lock();
        try {
            while (number != 2) {
                c2.await();
            }
            for (int i = 0; i < 10; i++) {
                System.out.println("BB");
            }
            number = 3;
            c3.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            reentrantLock.unlock();
        }
    }

    public void print15() {
        reentrantLock.lock();
        try {
            while (number != 3) {
                c3.await();
            }
            for (int i = 0; i < 15; i++) {
                System.out.println("CC");
            }
            number = 1;
            c1.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            reentrantLock.unlock();
        }
    }

}
```









## 十一  死锁

#### 11.1 什么是死锁？

死锁是指两个或两个以上的线程在执行过程中，因争夺资源而互相等待的线程，若无外力干涉，他们将一直持续下去。

![死锁](.\img\03\03_06.PNG)

**主要原因**

- 系统资源不足
- 程序执行顺序不合适
- 资源分配不当

**死锁示例**

```java
public class TestDeadLock {

    public static void main(String[] args) {
        final String lockA = "lockA";
        final String lockB = "lockB";

        new Thread(new DeadLockThread(lockA, lockB), "AA").start();
        new Thread(new DeadLockThread(lockB, lockA), "BB").start();
    }
}

class DeadLockThread implements Runnable {
    private String lockA;
    private String lockB;

    public DeadLockThread(String lockA, String lockB) {
        this.lockA = lockA;
        this.lockB = lockB;
    }

    @Override
    public void run() {

        synchronized (lockA) {

            System.out.println(Thread.currentThread().getName() + "  持有" + lockA + "，试图获得" + lockB);

            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (Exception e) {
                e.printStackTrace();
            }
            synchronized (lockB) {
                System.out.println(Thread.currentThread().getName() + "  持有" + lockB + "，试图获得" + lockA);
            }
        }
    }
}
```

**查看死锁**

通过jps，jstack可查看死锁



## 十二 CAS与原子类

#### 12.1 什么是CAS？CAS底层是如何实现？

比较交换（compare and swap）

在使用时，从变量的内存地址读取变量值，再与期望值进行比较，若一致，才能进行更新。

CAS通过调用sun.misc.Unsafe中方法实现，Unsafe类包含直接内存资源访问方法，能够用来操作内存。该类中方法多数为native方法，偏底层。程序中过度、不正确使用Unsafe类会使得程序出错的概率变大，使得Java这种安全的语言变得不再“安全”。

CAS操作底层实现依赖于CPU提供的特性指令，该指令的完成必须是连续的，执行过程中不会被中断，为原子指令，因此不会出现数据不一致情况，如x86的cmpxchg指令，均为非常轻量级的操作。

Java中的原子类如AtomicInteger，采用CAS实现乐观锁。

#### 12.2 CAS存在哪些问题？

- 自旋过多消耗CPU资源

  通常`CAS`失败，采用自旋重试机制。大多数情况，竞争短暂，一次重试，即可设置成功，但是难免存在意外情况，需要考虑自旋次数。

- ABA问题

  `CAS`操作更新变量时，需要比较内存变量值是否发生变化，未发生变化，再进行更新。若在此过程中，原先变量值为`A`，中途修改为`B`，又修改为`A`，`CAS`操作时，也能够更细成功，但是实际情况，变量值确实发生了变化。

- 只能保证对一个共享变量的原子操作

  对一个共享变量执行操作时，`CAS`能够保证原子操作，但是对多个共享变量操作时，`CAS`无法保证操作的原子性。

#### 12.3 什么是原子引用？

原子引用AtomicReference<V>，v可以是对象的引用，如自定义的Person对象，底层通过CAS比较的是对象引用的地址。

演示ABA问题

```java
    private static void testAtomicReference() {
        AtomicReference<Integer> atomicReference = new AtomicReference<>(111);

        new Thread(() -> {
            boolean result = atomicReference.compareAndSet(111, 113);
            System.out.println(Thread.currentThread().getName() + " " + result + " value = " + atomicReference.get());

            result = atomicReference.compareAndSet(113, 111);
            System.out.println(Thread.currentThread().getName() + " " + result + " value = " + atomicReference.get());
        }, "AAAA").start();

        new Thread(() -> {
            // 延时保证其它线程已经执完
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            boolean result = atomicReference.compareAndSet(111, 114);
            System.out.println(Thread.currentThread().getName() + " " + result + " value = " + atomicReference.get());
        }, "BBBB").start();

    }
```

打印结果

```java
AAAA true value = 113
AAAA true value = 111
BBBB true value = 114
```



#### 12.4 如何解决ABA问题？

一句话：添加版本号标签，可用工具类AtomicStampedReference

采用在变量上再贴一个标签，用来表示版本号，每次更新变量值时，标签值也会发生变化，用来判断是否发生过修改。在`Java1.5`，新增工具类`AtomicStampedReference`，即采用建立类似版本号`stamp`方式，解决`ABA`问题，确保`CAS`操作正确性。在`compareAndSet()`方法中，需要检查当前引用和当前标志与预期引用和预期标志是否相等，如果都相等，再通过原子操作来更新引用值和标签值。

**示例**

```java
    private static void testAtomicStampedReference() {
        AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(111, 1);

        new Thread(() -> {
            boolean result = atomicStampedReference.compareAndSet(111, 113, 1, 2);
            System.out.println(Thread.currentThread().getName() + " " + result + " value = " + atomicStampedReference.getReference());

            result = atomicStampedReference.compareAndSet(113, 111, 2, 3);
            System.out.println(Thread.currentThread().getName() + " " + result + " value = " + atomicStampedReference.getReference());
        }, "AAAA").start();

        new Thread(() -> {
            // 延时保证其它线程已经执完
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            boolean result = atomicStampedReference.compareAndSet(111, 114, 1, 4);
            System.out.println(Thread.currentThread().getName() + " " + result + " value = " + atomicStampedReference.getReference());
        }, "BBBB").start();
    }
```

结果

```java
AAAA true value = 113
AAAA true value = 111
BBBB false value = 111
```



#### 12.5 原子类AtomicInteger的使用？

如下Demo，提供10个线程，对变量`count、atomicInteger`进行自增1000次操作，执行完毕后，打印结果。

```java
public class TestAtomicInteger {

	private static AtomicInteger atomicInteger = new AtomicInteger();
	
	private static volatile int count = 0;
	
	public static void increase() {
		atomicInteger.incrementAndGet();
		count++;
	}
	
	public static void main(String[] args) {
		for (int i = 0; i < 10; i++) {
			new Thread(new Runnable() {
				@Override
				public void run() {
					for (int j = 0; j < 1000; j++) {
						increase();
					}
				}
			}).start();
		}
		
        while (Thread.activeCount() > 1) {
            Thread.yield();
        }
        
        System.out.println("atomicInteger = " + atomicInteger);
        System.out.println("count = " + count);
	}
}
```

多次操作，`AtomicInteger`修饰的变量结果一直为期望值10000。`int`型变量`count`得不到期望值。

```java
atomicInteger = 10000
count = 9758
```

问题原因是`count++`操作，为三个步骤组成，不具备原子性。并发情况下，在取count值、自增、刷新值至主内存中过程中，当最新值未刷新到主内存，其它线程已经读取了count存在内存中的旧值，就会出现错误。

#### 12.6 AtomicInteger实现原理？

为什么采用`AtomicInteger`修饰的变量不存在问题，其实现原理是什么？

查看源码`incrementAndGet()`方法，调用了`unsafe.getAndAddInt`方法实现。

```java
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }	
```

- 调用`sun.misc.Unsafe`类中方法实现

  `Unsafe`位于`sun.misc`包下，通过`Unsafe.getUnsafe()`安全的获取Unsafe实例， 然后再调用`getAndAddInt()`方法。`Unsafe`类包含直接内存资源访问方法，能够用来操作内存。该类中方法多数为`native`方法，偏底层。程序中过度、不正确使用`Unsafe`类会使得程序出错的概率变大，使得Java这种安全的语言变得不再“安全”。

- 变量`valueOffset`

  `AtomicInteger`对象中，用户设置值存储在变量`value`中，`valueOffset`用于存储变量`value`的内存偏移地址，使用CAS操作，需要获取变量内存地址。该值在类加载时进行赋值。

- `unsafe.getAndAddInt`方法

  传入参数var1，`AtomicInteger`对象的引用地址、var2偏移地址`valueOffset`，需要相加的加数值var4。

  `getAndAddInt`方法实现代码

  ```java
  public final int getAndAddInt(Object var1, long var2, int var4) {   
      int var5;     
      do {          
          var5 = this.getIntVolatile(var1, var2);   
      } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));    
      return var5;   
  }
  ```

  通过`getIntVolatile`方法从内存偏移地址中获取期望值`var5`，`var5 + var4`计算待更新值，通过`while`循环，调用`compareAndSwapInt`方法，用对象当前值与期望值var5，不断进行比较，若相同则更新，返回true，循环取反，循环结束；若不同，返回false，继续循环。getAndAddInt()方法更新成功后返回更新前的值。由于`incrementAndGet()`方法会返回增加后的值，所以在调用完`getAndAddInt`方法后，会再`+1`。而`getAndIncrement`方法不会`+1`。

#### 12.7 为什么AtomicInteger中采用CAS，不用synchronized？

synchronized通过加锁，一次只能有一个线程访问，能够保证一致性，但并发性下降。

通过采用CAS，无需加锁，采用自旋，不断比较，直到成功为止，能够保证一致性，也能保证并发性。

假设有两个线程A，B，同时执行AtomicInteger中的getAndAddInt方法，假设内存中value值为5

1）基于JMM模型，value值5存放在主内存，线程A和B的工作内存中存有变量3的副本

2）当线程A，通过getIntVolatile拿到value值为5，线程此时被挂起

3）线程Ｂ执行，同样通过getIntVolatile拿到value值为5，调用compareAndSwapInt方法，再次从内存中读取value值与期望值5比较，相同，则可以进行更新，更新为8，执行完毕

４）此时线程Ａ调度回来接着执行，此时主内存中的value值已经被改变为8，而第一次获取到期望值仍为3，再调用compareAndSwapInt方法时，还是会从内存中再次读取一次value值为8，与期望值不符，compareAndSwapInt返回false，继续自旋，直至成功。

