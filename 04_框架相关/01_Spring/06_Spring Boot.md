## Spring Boot

## 1.  Spring Boot设计核心思想？

核心：简化Spring应用的初始搭建以及开发过程。

1）内嵌web服务器tomcat，直接运行一个main方法，就能够将程序运行起来，简化了运行流程

2）自动配置，提供了配置应用程序和框架所需要的基本配置，比如整合MyBatis，引入一个mybatis-spring-boot-starter，Spring Boot能够自动引入一些Jar包，同时会自动完成mybatis的一些配置和定义，不需要再手工一项项去配置，简化了配置流程





## 2. Spring Boot、Spring MVC 和 Spring 有什么区别？

SpringMVC 是Spring的一个模块，基于Spring的一个 MVC 框架；通过DispatcherServlet, ModelAndView 和 View Resolver，开发web应用。

Spring 框架就像的基础是Spring 的ioc和 aop，ioc 提供了依赖注入的容器， aop解决了面向切面编程，然后在此两者的基础上实现了其他延伸产品的高级功能。

SpringBoot，简化Spring应用的初始搭建以及开发过程。



## 3. SpringBoot启动流程

程序入口为

```java
@SpringBootApplication
public class Application {
  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}
```

### @SpringBootApplication注解

SpringBootApplication注解作用，



### SpringApplication.run(Application.class, args);















