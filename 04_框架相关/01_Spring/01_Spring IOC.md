# Spring框架知识

## 一 IOC容器

### 1.1 说说对IOC容器的理解？

IOC，称为控制反转，为一种设计思想。

传统创建对象可以通过new对象直接实例化，但是通常一个Service对象可能存在复杂的依赖，使用该对象时需要去维护复杂依赖关系，使得对这个对象的引用变得复杂。

IOC容器通过控制反转，依赖注入来解决这一问题，实现类与类之间的解耦合。比如，

```java
@Service
public class MyServiceImpl implements MyService() {}
// 使用时需要替换MyServiceImpl为NewMyServicveImpl,去掉原来@Service注解
@Service
public class NewMyServicveImpl implements MyService() {}
```

应用启动时，Spring IOC容器，会加载xml配置文件或者注解，实例化一些bean对象，再根据详细配置，去初始化对象或者处理bean对象之间的依赖关系，进行依赖注入，从而完成对应用中使用到对象的创建和管理。

底层核心是通过反射技术。

**总结**

统一对象的创建，自动维护管理对象（管理对象之间依赖关系），使类与类之间解耦。底册实现通过反射技术，可以采用配置XML文件或者注解的方式实现，降低了编码和维护的复杂度。

### 1.2 Bean的构建方法有哪些？

Spring中Bean的创建，主要有四种方式

1. xml中基于ClassName构建

```java
<bean class="com.ttsummer.testSpring"></bean>
```

2. 基于构造方法创建

```java
<bean class="com.ttsummer.testSpring">
	<constructor-arg name="name" type="java.lang.String" value="tttt"/>
    <constructor-arg index="1" type="java.lang.String" value="sex" />
</bean>
```

3. 静态工厂方法创建



4. FactoryBean创建



### 1.3 Bean的作用域包含哪些？

通过配置参数scope可配置Bean的作用域，包含

- scope=“singleton”，单例，默认为单例
- scope=“prototype”，多例的，每次请求都会创建一个新的Bean实例

还包含其它request、session等参数

- scope=“request”，每次http请求都会创建一个新的Bean，但仅在该次HTTP request中有效，请求完成后，Bean会失效
- scope=“session”，每个Session中创建一个Bean，Session过期后，Bean实现

### 1.4 Bean的生命周期？

Bean对象生命周期包含创建、初始化、销毁。可以通过配置参数 

- init-method，bean被初始化时执行
- destroy-method ，容器销毁前执行

```java
<bean class="com.ttsummer.spring.HelloSpring" init-method="init" destroy-method="destroy"></bean>
```

- 创建阶段

  1. Spring启动，查找并加载需要被Spring管理的bean，进行Bean的实例化
  2. Bean实例化后对将Bean的引入和值注入到Bean的属性中
  3. 如果Bean实现了BeanNameAware接口的话，Spring将Bean的Id传递给setBeanName()方法
  4. 如果Bean实现了BeanFactoryAware接口的话，Spring将调用setBeanFactory()方法，将BeanFactory容器实例传入
  5. 如果Bean实现了ApplicationContextAware接口的话，Spring将调用Bean的setApplicationContext()方法，将bean所在应用上下文引用传入进来。
  6. 如果Bean实现了BeanPostProcessor接口，Spring就将调用他们的postProcessBeforeInitialization()方法。
- 初始化阶段
  1. 如果Bean 实现了InitializingBean接口，Spring将调用他们的afterPropertiesSet()方法。类似的，如果bean使用init-method声明了初始化方法，该方法也会被调用
  2. 此时，Bean已经准备就绪，可以被应用程序使用了。他们将一直驻留在应用上下文中，直到应用上下文被销毁。

- 销毁

  

#### 1. 实例化Bean

对于BeanFactory容器，当客户向容器请求一个尚未初始化的bean时，或初始化bean的时候需要注入另一个尚未初始化的依赖时，容器就会调用createBean进行实例化。 
对于ApplicationContext容器，当容器启动结束后，便实例化所有的bean。 
容器通过获取BeanDefinition对象中的信息进行实例化。并且这一步仅仅是简单的实例化，并未进行依赖注入。 
实例化对象被包装在BeanWrapper对象中，BeanWrapper提供了设置对象属性的接口，从而避免了使用反射机制设置属性。

#### 2. 填充对象属性（依赖注入）

实例化后的对象被封装在BeanWrapper对象中，并且此时对象仍然是一个原生的状态，并没有进行依赖注入。 
紧接着，Spring根据BeanDefinition中的信息进行依赖注入。 
并且通过BeanWrapper提供的设置属性的接口完成依赖注入。

#### 3. 注入Aware接口

紧接着，Spring会检测该对象是否实现了xxxAware接口，并将相关的xxxAware实例注入给bean。

#### 4. BeanPostProcessor(前置、后置处理器)

当经过上述几个步骤后，bean对象已经被正确构造，但如果你想要对象被使用前再进行一些自定义的处理，就可以通过BeanPostProcessor接口实现。 
该接口提供了两个函数：

- postProcessBeforeInitialzation( Object bean, String beanName ) 
  当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。 
  这个函数会先于InitialzationBean执行，因此称为前置处理。 
  所有Aware接口的注入就是在这一步完成的。
- postProcessAfterInitialzation( Object bean, String beanName ) 
  当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。 
  这个函数会在InitialzationBean完成后执行，因此称为后置处理。

#### 5. InitializingBean与init-method

当BeanPostProcessor的前置处理完成后就会进入本阶段。 
InitializingBean接口只有一个函数：

- afterPropertiesSet()

这一阶段也可以在bean正式构造完成前增加我们自定义的逻辑，但它与前置处理不同，由于该函数并不会把当前bean对象传进来，因此在这一步没办法处理对象本身，只能增加一些额外的逻辑。 
若要使用它，我们需要让bean实现该接口，并把要增加的逻辑写在该函数中。然后Spring会在前置处理完成后检测当前bean是否实现了该接口，并执行afterPropertiesSet函数。

当然，Spring为了降低对客户代码的侵入性，给bean的配置提供了init-method属性，该属性指定了在这一阶段需要执行的函数名。Spring便会在初始化阶段执行我们设置的函数。init-method本质上仍然使用了InitializingBean接口。

#### 6. DisposableBean和destroy-method

和init-method一样，通过给destroy-method指定函数，就可以在bean销毁前执行指定的逻辑。



#### 总结

class文件 

转化为BeanDefinition

通过BeanFactory构建

执行BeanFactoryPostProcessor，能够拿到beanFactory，来修改Bean

然后是实例化Bean，（此时会提前曝光，将原始对象A放到一个map中）

填充属性（可能会依赖于其它bean B，会构造一个其它bean B，但是bean可能会依赖于A，出现循环依赖，会从缓存中获取，查看是否存在A，存在A完成填充）

执行实现了Aware接口的方法，init

初始化，执行BeanPostProcessor（前置，后置处理器）

将对象放入单例池（singleObjects， beanNmae：bean对象）





```
Bean工厂后置处理器
BeanFactoryPostProcessor
Bean后置处理器
BeanPostProcessor
```





[Spring Bean生命周期](https://www.bilibili.com/video/BV1KC4y1t7x4?p=3&spm_id_from=pageDriver)

[深究Spring中Bean的生命周期](https://www.cnblogs.com/javazhiyin/p/10905294.html)

[Spring中Bean的生命周期是怎样的？](https://www.zhihu.com/question/38597960)





### 1.5 Bean中的依赖关系是如何处理的？及如何处理循环依赖？

循环依赖是指，对象A依赖于B，B依赖于C，C依赖于A，类似这样循环依赖。

Spring在获取对象A时，通过提前曝光，会将A加入正在创建Bean的缓存中。在创建A过程中，发现依赖于，需要创建B，创建B时，又发现依赖于A，此时的A已经被提前曝光加入了正在创建的bean的缓存中，则无需创建新的的ClassA的实例，直接从缓存中获取即可。从而解决循环依赖问题。

Spring只能解决Setter方法注入的单例bean之间的循环依赖



主要通过二级，三级缓存来解决循环依赖。

如果没有AOP，使用二级缓存即可解决循环依赖

存在AOP，需要结合三级缓存来解决循环依赖。



一级缓存：

singletonObjects：缓存某个beanName对应的经过了完整生命周期的bean

二级缓存：

earlySingletonObjects：缓存提前拿原始对象进行了AOP之后得到的代理对象，原始对象还没有进行属性注入和后续的BeanPostProcessor等生命周期。

只有出现了循环依赖，才会有值

三级缓存：

singletonFactories：缓存的是一个ObjectFactory，主要用来生成原始对象进行了AOP之后得到的代理对象，在每个Bean的生成过程中，会提前暴露一个工厂，没有出现循环依赖本bean，那个这个工厂不使用，本bean按照自己的生命周期执行，执行完成后把本bean放入一级缓存singletonObjects中。如果出现了循环依赖了本bean，则另外那个bean执行ObjectFactory提交到一个AOP之后的代理对象。如果无需AOP，则直接得到一个原始对象

其它

还存在一个缓冲earlyProxyReference，用来记录某个原始对象是否进行过了AOP



#### 流程

class文件 

转化为BeanDefinition

通过BeanFactory构建

执行BeanFactoryPostProcessor，能够拿到beanFactory，来修改Bean

然后是实例化Bean，（此时会提前曝光，将原始对象A放到一个map中）

填充属性（可能会依赖于其它bean B，会构造一个其它bean B，但是bean可能会依赖于A，出现循环依赖，会从缓存中获取，查看是否存在A，存在A完成填充）

执行实现了Aware接口的方法，init

初始化，执行BeanPostProcessor（前置，后置处理器）

将对象放入单例池（singleObjects， beanNnae：bean对象）









### 1.6 Bean的加载机制？

可以配置Bean的加载时间，设置属性lazy-init

- true，懒加载，即延迟加载。当真正使用对象的时候才创建对象，启动较快，但不易发现错误。
- false，非懒加载，容器启动时即创建对象，容器启动过程即可发现程序错误。
- default，默认，采用default-lazy-init 中指定值，如果default-lazy-init  没指定就是false

### 1.7 Bean的依赖注入（DI）？

一个Bean依赖于另外一个Bean，传统可以通过外部传入，内部构建。

依赖注入主要解决，由于应用程序依赖于某个对象，IOC能够自动将应用程序依赖的对象注入进容器中。

Spring中如下四种方式

1. set方法注入

对象中需要创建引入对象的set方法。

```java
    <bean id="HelloPerson" class="com.ttsummer.entity.Person">
        <!--set方法注入-->
        <property name="animal" ref="helloAnimal"/>
    </bean>
```

2. 构造方法注入

```java
    <bean id="HelloPerson" class="com.ttsummer.entity.Person">
        <constructor-arg ref="helloAnimal"/>
    </bean>
```

3. 自动注入

```java
    <bean id="HelloPerson" class="com.ttsummer.entity.Person">
        <constructor-arg ref="helloAnimal"/>
    </bean>
```
byName：被注入bean的id名必须与set方法后半截匹配
byType：查找所有的set方法，将符合参数类型的bean注入

4. 通过注解（Autowired）

```java
    @Autowired
    private Animal animal;
```

@Autowired 和 @Resource 区别  
共同点  
@Resource和@Autowired都可以作为注入属性的修饰，在接口仅有单一实现类时，两个注解的修饰效果相同，可以互相替换，不影响使用。  
不同点  
1、@Resource是JDK原生的注解，@Autowired是Spring2.5 引入的注解  
2、@Resource有两个属性name和type。Spring将@Resource注解的name属性解析为bean的名字，而type属性则解析为bean的类型。所以如果使用name属性，则使用byName的自动注入策略，而使用type属性时则使用byType自动注入策略。如果既不指定name也不指定type属性，这时将通过反射机制使用byName自动注入策略。 @Autowired只根据type进行注入，不会去匹配name。如果涉及到type无法辨别注入对象时，那需要依赖@Qualifier或@Primary注解一起来修饰。  


### 1.8 Bean是如何被构建的？

**一句话概括**

在spring.xml文件中配置对Bean的描述配置，BeanFactory 会读取这些配置然后生成对应的Bean。

**中间过程**

**Bean的存放位置**（BeanDefinition 、BeanDefinitionRegistry ）

1. 从xml文件中读取Bean描述信息，存放到BeanDefinition 中

ioc 实现中xml 中描述的Bean信息都将保存至BeanDefinition 对象中，其中xml 中bean 与BeanDefinition一对一

2. 将bean的id和name参数，通过BeanDefinitionRegistry进行注册，

 xml中bean的id和name属性没有包含在BeanDefinition中，将ID 作为当前Bean的存储key注册到了BeanDefinitionRegistry 注册器中，name作为别名key 注册到了 AliasRegistry 注册中心。其最后都是指向其对应的BeanDefinition，存在ConcurrentHashMap中。

![image-20210405235435721](.\img\04_01_06.png)

**谁来主导读取并装载Bean**(BeanDefinitionReader)

3. 通过BeanDefinitionReader(Bean定义读取器)，读取Bean

BeanDefinition 中存储了Xml Bean信息，而BeanDefinitionRegister基于ID和name保存了Bean的定义。

通过BeanDefinitionReader来完成从xml中的Bean，读取到BeanDefinition 中，然后注册到BeanDefinitionRegister 中。

![BeanDefinitionReader](.\img\04_01_01.png)

BeanDefinitionReader中包含方法loadBeanDefinitions，用于读取Bean，并注册至注册器。

![](.\img\04_01_02.png)

**Bean信息已被读取到内存中，谁来处理Bean中配置信息**

4. Bean已被读取，通过bean 工厂，Beanfactory来处理构建Bean

![BeanFactory方法](.\img\04_01_03.png)

当调用getBean()方法，Beanfactory就会对触发对Bean的创建

Bean创建时序图

![Bean创建时序图](.\img\04_01_04.png)

流程

1. 调用getBean()获取IOC容器中Bean
2. 通过Bean工厂（BeanFactory）调用getSingleton()方法，从Bean缓存singletonObjects（为一个ConcurrentHashMap）中尝试获取Bean（创建的单例Bean都会被缓存起来）

获取过程中用到了双重校验来保证正确获取

```java
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

3. 若缓存中不存在，则会进行重新创建；若存在，返回Object
4. 获取当前Bean是否存在依赖，若存在，先进行创建
5. 需要判断参数scope为singleton还是Prototype，然后分别走不同分支，调用createBean()方法创建Bean
6. 单例创建完后加入到缓存中，
7. 最后通过反射创建Bean













### 1.9 Bean工厂BeanFactory与上下文ApplicationContext区别？

![](.\img\04_01_05.png)

ApplicationContext接口是由BeanFactory接口派生而来，提供了BeanFactory所有的功能，同时还实现了很多接口，通过该接口能够调用实现了这些接口的类中方法，ClassPathXmlApplicationContext即为。

ApplicationContext扩展了如下功能

1. MessageSource, 提供国际化的消息访问
2. 资源访问，如URL和文件
3. 事件传播，实现了ApplicationListener接口的bean
4. 载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层



### 1.10 将一个类声明为Spring的Bean的注解有哪些？  

- @Component，用于标注任意类为Spring组件，当不明确该类为哪一层，可以该注解描述
- @Repository，持久层Dao层，用于数据库相关操作
- @Service，对应服务层
- @Controller，对应控制层

### 1.11 @Component 与@Bean的区别？

- 作用对象不同，@Bean作用于方法，@Component作用于类

- 作用不同，@Component，通常用于自动扫描及自动装配到Spring容器中

   @Bean，用于显式声明单个bean，告诉Spring此为某个类的实例，可以将第三方类变成组件，通过和@Configuration注解搭配使用



### 1.12 Spring中的Bean是单例的吗？如果是单例如何保证线程安全？


默认是单例，可以通过scope进行作用域配置。

1. 在 `@Controller/@Service` 等容器中，默认情况下，scope值是单例-singleton的，也是线程不安全的。
2. 尽量不要在`@Controller/@Service` 等容器中定义静态变量，不论是单例(singleton)还是多实例(prototype)他都是线程不安全的。
3. 默认注入的Bean对象，在不设置`scope`的时候他也是线程不安全的。
4. 一定要定义变量的话，用`ThreadLocal`来封装，这个是线程安全的。
5. 


