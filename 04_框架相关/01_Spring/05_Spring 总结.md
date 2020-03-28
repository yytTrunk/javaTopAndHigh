## Spring总结

###1. 谈谈对Spring的理解？





### 2.Spring中用到的设计模式？

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

