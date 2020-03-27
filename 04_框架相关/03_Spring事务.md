# Spring事务

## 一 Spring中事务管理

### 1.1	什么是事务？

事务管理是指需要保证多线程并发环境下，对数据库的操作依然能够保证正确性。即需要保证数据库事务的四大特性ACID，原子性、一致性、隔离性和持久性

- 原子性（atomicity）
- 一致性
- 隔离性
- 持久性

数据库中提供了四种隔离级别。

Spring中的事务本质也是对数据库事务的支持，没有数据库事务的支持，Spring本身也难以提供事务功能。



### 1.2 Spring中事务？

Spring中事务在数据库事务基础上进行了封装，并进行了扩展。

1. 支持了原有数据库事务的隔离级别
2. 加入了事务传播
3. 提供了声明式事务







https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#aop-proxying)