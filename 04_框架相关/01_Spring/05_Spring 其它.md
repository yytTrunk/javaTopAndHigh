# Spring总结

###1. 谈谈对Spring的理解？

Spring核心，IOC（控制反转，依赖注入）和AOP。

应用启动时，Spring容器会去读取XML文件或者根据注解，去创建Bean对象实例（基于反射技术），由于对象实例中可能存在依赖关系，会进行依赖注入。

同时存在以下配置方法



### 2.Spring中用到了哪些设计模式？

- 工厂设计模式

使用工厂模式通过BeanFactory、ApplicationContext创建Bean对象

![](.\img\04_01_05.png)

- 单例模式

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

Spring AOP实现

### 



3. Spring中常用注解有哪些？

微信文章