# Spring AOP

## 一 AOP

### 1.1 说说对AOP的理解？

介绍AOP主要思想是什么

应用于哪些场景

核心概念

Spring中实现AOP主要采用什么技术实现（代理，需要将把代理逻辑织入到目标对象上，生成一个代理对象）



### 1.2 AOP是什么？

面向切面编程

通常OOP开发是从上而下的，会产生一些横切性的问题，比如日志记录，权限验证等，这些与主业务逻辑关系不大，但是会散落出现在代码的各个部分，使代码显得复杂并难以维护。

AOP即能够很好处理这样横切性问题，AOP的思想就是把横向插入在业务逻辑上的代码与主逻辑代码分开，通过更简单的方式达到目的，从而降低耦合度，减少重复代码，增强代码逻辑，提高了代码重用性和开发效率。

底层实现主要基于动态代理技术。



### 1.3 AOP的应用场景？AOP使用流程？

日志记录、权限验证、事务管理、异常处理等

例如事务管理流程，开启事务->执行SQL->提交事务或者回滚事务

此时可以通过面向切面编程AOP实现。首先声明一个切面Aspect，指明需要切入的点（ponitcut，执行SQL代码位置），增强的方法（前置开启事务、后置提交或回滚事务），执行原来代码时，会通过动态代理生成代理类，将切面中增强的逻辑代码，织入进切入点中（原来代码中），执行原来对象时会调用代理对象，此时代理对象中已经增强了方法，即可达到效果。

代码流程：定义切面（Aspect）->描述切入点（pointcut）->添加增强方法（advice）->注入目标对象时切面织入进目标对象，生成代理对象->实际调用目标对象时为代理对象->原始对象代码和增加后的代码已融合

### 1.4 Spring AOP中核心概念？

**1、横切关注点**

对哪些方法进行拦截，拦截后怎么处理，这些关注点称之为横切关注点

**2、切面（aspect）**

类是对物体特征的抽象，切面就是对横切关注点的抽象

**3、连接点（joinpoint）**

就是需要被拦截到的点，

因为Spring只支持方法类型的连接点，所以在Spring中连接点指的就是被拦截到的方法，实际上连接点还可以是字段或者构造器

**4、切入点（pointcut）**

切入连接点，就是切入拦截的点

切入点和连接点可以一起看

**5、通知（advice）**

所谓通知指的就是指拦截到连接点之后要执行的代码，通知分为前置、后置、异常、最终、环绕通知五类

- Before
- After
- After throwing
- After finally
- Around advice

**6、目标对象**（target）

代理的目标对象，原始对象

**7、织入（weave）**

把代理逻辑加入到目标对象上的过程叫做织入

将切面应用到目标对象并导致代理对象创建的过程

**8、代理对象（AOP proxy）**

代理对象，包含了原始对象的代码和增加后的代码的那个对象



### 1.4 Spring AOP实现的主要技术？CGLIB 与JDK代理区别？

前提Spring AOP是基于代理实现。

如果代理的对象实现了某个接口，Spring AOP会使用JDK Proxy，去创建对象

对没有实现接口的对象，无法使用JDK Proxy去代理，而是通过CGLib Proxy去创建对象，

JDK代理和CGLib代理都是在运行时期织入，都是在对象初始化时期织入。

> JDK 动态代理机制只能对接口进行代理，其原理是动态生成一个代理类，这个代理类实现了目标对象的接口，将需要增强的代码插入到对应方法中。目标对象和代理类都实现了接口，但是目标对象和代理类的 Class 对象是不一样的，所以两者是没法相互赋值的。

> CGLIB 是对目标对象本身进行代理，所以无论目标对象是否有接口，都可以对目标对象进行代理，其原理是使用字节码生成工具在内存生成一个继承目标对象的代理类，然后创建代理对象实例。由于代理类的父类是目标对象，所以代理类是可以赋值给目标对象的，自然如果目标对象有接口，代理对象也是可以赋值给接口的

> JDK 使用反射机制调用目标类的方法，CGLIB 则使用类似索引的方式直接调用目标类方法，所以 JDK 动态代理生成代理类的速度相比 CGLIB 要快一些，但是运行速度比 CGLIB 低一些，并且 JDK 代理只能对有接口的目标对象进行代理。

**核心思想流程**

无论JDK代理还是CGULIB代理，都是通过生成一个代理对象，在该对象会插入增强代码，同时再调用目标对象的代码。

存在接口时，生成一个实现了该接口的代理对象，调用代理的方法时，先走到实现的代理对象的invok方法中，该方法中，进行代码增强，然后会找到原来对象的对应方法，再调用目标对象的方法，完成代码增强。

当为一个类时，会继承目标对象，生成代理对象，在调用目标对象时，会去调用代理对象的intercept方法，进行拦截，插入需要增强的方法，同时再根据方法名，找到目标对象的方法，进行调用，即完成代码增强。

[JDK动态代理](https://www.cnblogs.com/muscleape/p/9018302.html )

[CGLIB动态代理](https://www.cnblogs.com/muscleape/p/9018308.html)

**JDK 代理为什么只能是实现接口的类，CGLIB是基于类继承？**

因为JDK代理生成的代理类，会去继承一个Proxy类，由于Java是单继承，所以只能是基于接口的类才是JDK 代理。



Spring AOP如果代理的类存在接口，优先使用JDK动态代理，否则使用CGLIB动态代理。

### 1.5 Spring AOP 与AspectJ的关系？

AOP只是一种概念，Spring AOP和AspectJ AOP均是AOP的实现。

Spring AOP是基于运行时动态代理实现，而Apectj是编译时增强，是基于字节码操作，能够利用专门的编译器用来生成遵守Java字节编码规范的Class文件。

Spring中只是借助了AspectJ的注解，其底层实现还是自己的。



### 1.6 Spring AOP的使用方法？

两种，一种是基于XML配置文件，一种是基于注解。

- 基于XML

通过aop:config 命名空间，通过aop:aspect、aop:pointcut、aop:before method等参数来配置

- 基于注解

使用@AspectJ注解，但是SpringAop借助了AspectJ的注解，其底层实现还是自己的。

```java
步骤1
使用xml配置启用@AspectJ
<aop:aspectj-autoproxy/>

使用注解启用@AspectJ支持
@Configuration
@EnableAspectJAutoProxy

步骤2 
声明一个Aspect，定义成Bean由Spring管理
@Component
@Aspect
public class NotVeryUsefulAspect {
    @Pointcut("@annotation(com.xyz.Transactional)")
	public void ponitCut() {
		
	}
	@Before("ponitCut()")
	public void beforeAdvice() {
		System.out.println("begin transaction");
	}
	
	@After("ponitCut()")
	public void afterAdvice() {
		System.out.println("end transaction");
	}
}

步骤3 
声明一个pointCut切入点，包含一个切入点表达式，用于指定对切入的具体方法
@Pointcut("execution(public * com.xyz.dao.*.*())")//匹配com.xyz.dao包下的任意接口和类的public 无方法参数的方法
@Pointcut("within(com.xyz.dao.*)")//匹配com.xyz.dao包中的任意方法
@Pointcut("@annotation(com.xyz.Transactional)")//匹配带有com.xyz.Transactional注解的方法
等等还包含其它表达式@target  @args

步骤四
声明通知，拦截到连接点之后要执行的代码，通知分为前置、后置、异常、最终、环绕通知五类
- Before
- After
- After throwing
- After finally
- Around advice
```

Spring切面粒度最小是达到方法级别，而execution表达式可以用于明确指定方法返回类型，类名，方法名和参数名等与方法相关的信息。使用execution基本可以满足需求。



### 关于Spring AOP参考文档，官网

[Spring-framework](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#aop-proxying)