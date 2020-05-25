# Spring事务

## 一 Spring中事务管理

### 1.1	什么是事务？

事务管理是指需要保证多线程并发环境下，对数据库的操作依然能够保证正确性。即需要保证数据库事务的四大特性ACID，原子性、一致性、隔离性和持久性

- 原子性（atomicity）
- 一致性（consistency）
- 隔离性（isolation）
- 持久性（durability）

数据库中提供了四种隔离级别。

Spring中的事务本质也是对数据库事务的支持，没有数据库事务的支持，Spring本身也难以提供事务功能。



### 1.2 Spring中事务的实现原理？

Spring中事务在数据库事务基础上进行了封装，并进行了扩展。

1. 支持了原有数据库事务的隔离级别
2. 加入了事务传播
3. 提供了声明式事务

**Spring中使用事务**

可以通过使用注解@Transaction，实现事务，加上该注解后，Spring会基于AOP思想，在该方法执行之前，去开启事务，执行完成之后，根据执行结果，来确定是提交事务还是回滚事务。



### 1.3 Spring中事务的传播机制？

通过配置参数，可以设置事务传播类型

- #### PROPAGATION_REQUIRED（支持当前事务）（常用默认）

如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。

```java
// Propagation.REQUIRED
// 如果当前方法A没有事务，就新建一个事务，
// 方法A调用方法B，如果已经存在一个事务中，加入到该事务中。
@Transactional(propagation = Propagation.REQUIRED)
public void addUser() {
    addRelation();
}
```

- #### PROPAGATION_SUPPORTS（支持当前事务）

支持当前事务，如果当前没有事务，就以非事务方式执行；当前存在事务，就加入到该事务中。

- #### PROPAGATION_MANDATORY（支持当前事务）

使用当前的事务，如果当前没有事务，就抛出异常。

如果方法B没有事务，直接调用方法B就会抛出异常；如果方法A存在事务，去调用方法B，则会加入该事务中。

- #### PROPAGATION_REQUIRES_NEW （不支持当前事物）

新建事务，如果当前存在事务，把当前事务挂起。

方法A调用方法B，方法B也会新建一个事务，B事务与A事务互不影响。

**适用于** 该方法业务逻辑不能受主业务逻辑影响，如日志记录，不能因为主业务执行失败，需要回滚，导致日志记录也去回滚，使得日志无法记录。

- #### PROPAGATION_NOT_SUPPORTED（不支持当前事物）

以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。

方法A调用方法B，如果B为该级别事务，B以非事务方式执行，执行失败不会回滚，方法A也不会受影响，不会回滚。

**适用于** 当前方法对主代码逻辑影响不大的情况，如发优惠券、发短信通知等

- #### PROPAGATION_NEVER（不支持当前事物）

以非事务方式执行，如果当前存在事务，则抛出异常。

- #### PROPAGATION_NESTED（嵌套事务）

方法A调用方法B，方法A事务回滚，B也会回滚，但是如果B回滚，A不会回滚



## 参考

https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#aop-proxying)