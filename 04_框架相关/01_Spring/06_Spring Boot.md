## Spring Boot

### 1.  Spring Boot设计核心思想？

核心：简化Spring应用的初始搭建以及开发过程。

1）内嵌web服务器tomcat，直接运行一个main方法，就能够将程序运行起来，简化了运行流程

2）自动配置，提供了配置应用程序和框架所需要的基本配置，比如整合MyBatis，引入一个mybatis-spring-boot-starter，Spring Boot能够自动引入一些Jar包，同时会自动完成mybatis的一些配置和定义，不需要再手工一项项去配置，简化了配置流程

