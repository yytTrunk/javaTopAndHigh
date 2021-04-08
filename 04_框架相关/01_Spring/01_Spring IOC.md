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

### 1.5 Bean的加载机制？

可以配置Bean的加载时间，设置属性lazy-init

- true，懒加载，即延迟加载。当真正使用对象的时候才创建对象，启动较快，但不易发现错误。
- false，非懒加载，容器启动时即创建对象，容器启动过程即可发现程序错误。
- default，默认，采用default-lazy-init 中指定值，如果default-lazy-init  没指定就是false

### 1.6 Bean的依赖注入（DI）？

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
byType：查找所有的set方法，将符合符合参数类型的bean注入

4. 通过注解（Autowired）

```java
    @Autowired
    private Animal animal;
```

需要在xml文件中开启

### 1.7 Bean是如何被构建的？

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



### 1.8 Bean中的依赖关系是如何处理的？及如何处理循环依赖？

循环依赖是指，对象A依赖于B，B依赖于C，C依赖于A，类似这样循环依赖。

Spring在获取对象A时，通过提前曝光，会将A加入正在创建Bean的缓存中。在创建A过程中，发现依赖于，需要创建B，创建B时，又发现依赖于A，此时的A已经被提前曝光加入了正在创建的bean的缓存中，则无需创建新的的ClassA的实例，直接从缓存中获取即可。从而解决循环依赖问题。

Spring只能解决Setter方法注入的单例bean之间的循环依赖

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


