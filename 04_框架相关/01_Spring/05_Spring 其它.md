# Spring总结

###1. 谈谈对Spring的理解？

Spring核心，IOC（控制反转，依赖注入）和AOP。

应用启动时，Spring容器会去读取XML文件或者根据注解，去创建Bean对象实例（基于反射技术），由于对象实例中可能存在依赖关系，会进行依赖注入。同时查看是否存在AOP，使用动态代理生成Bean对象。

同时具备以下扩充配置方法

Spring 中Bean的生命周期包含创建、初始化、销毁。

1）创建Bean时，会将xml中描述Bean信息都将保存至BeanDefinition 对象，然后通过BeanDefinitionRegistry进行注册，注册过程即保存Bean信息到内存中，为一个ConcurrentHashMap对象。

2）解析加载Bean，依据xml文件中对Bean的描述信息，进行解析加载loadBeanDefinitions。

3）Bean的依赖注入，会判断这个Bean是否存在依赖关系，若存在也需要进行注入，可以通过set方法注入、构造方法注入、自动注入和注解。如果存在AOP，需要生成动态代理对象。

4）Bean的加载机制，支持懒加载、非懒加载。

5）Bean的**创建**，调用getBean()方法后，通过Beanfactory，使用反射进行创建。

6）Bean的作用域，创建Bean时，会根据参数scope值，判断为单例、多例、request还是Session。

7）Bean构建好后，判断是否存在后置处理器BeanPostProcessor，可以在初始化前后，执行自定义方法

接着，会查看是否显式指定初始化方法init-method，进行**初始化**

8）创建完成后，Bean会长期存活在Spring 的IOC容器中

9）Bean使用后，可以进行**销毁**，销毁前会判断是否指定了destroy-method，销毁前会执行

### 2.Spring中用到了哪些设计模式？

- 工厂模式

Spring IOC就相当于一个大的工厂，将所有的Bean实例都方法在Spring容器中，当需要使用哪个Bean时，就可以从Spring容器中取。

使用工厂模式通过BeanFactory、ApplicationContext创建Bean对象

![](.\img\04_01_05.png)

- 单例模式

Spring中Bean作用域默认为单例。

Spring IOC中当getBean()时，会先从singletonObjects（为ConcurrentHashMap）一级缓存池中，去尝试获取单例Bean，此时通过双重校验保证获取正确，单例模式。

singletonObjects（为一个ConcurrentHashMap）中尝试获取Bean

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

- 代理模式

Spring AOP能够对一些类织入一些增强代码，生成动态代理对象，调用者访问目标对象时，会先经过代理对象，此时会执行增强代码，然后再执行目标对象中方法。

以上即为代理模式的体现，使用动态代理对象，去代理目标对象，同时加入一些增强代码。

### 









