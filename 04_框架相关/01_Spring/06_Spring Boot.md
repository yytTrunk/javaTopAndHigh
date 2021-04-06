## Spring Boot

## 1.  Spring Boot设计核心思想？

核心：简化Spring应用的初始搭建以及开发过程。

1）内嵌web服务器tomcat，直接运行一个main方法，就能够将程序运行起来，简化了运行流程

2）自动配置，提供了配置应用程序和框架所需要的基本配置，比如整合MyBatis，引入一个mybatis-spring-boot-starter，Spring Boot能够自动引入一些Jar包，同时会自动完成mybatis的一些配置和定义，不需要再手工一项项去配置，简化了配置流程





## 2. Spring Boot、Spring MVC 和 Spring 有什么区别？

SpringMVC 是Spring的一个模块，基于Spring的一个 MVC 框架；通过DispatcherServlet, ModelAndView 和 View Resolver，开发web应用。

Spring 框架就像的基础是Spring 的ioc和 aop，ioc 提供了依赖注入的容器， aop解决了面向切面编程，然后在此两者的基础上实现了其他延伸产品的高级功能。

SpringBoot，简化Spring应用的初始搭建以及开发过程。







## 3. SpringBoot自动装配流程

@import作用

1.定义一个类

2.进行整合

3.导入一个选择器，自动装配



SpringBoot启动注解

```
第一层
@SpringBootApplication
第二层
@SpringBootConfiguration   // Spring的配置类
@EnableAutoConfiguration
第三层
@Import(AutoConfigurationImportSelector.class)
```

AutoConfigurationImportSelector作用

```java
/*
  所有的配置都存放在configurations中，
  而这些配置都从getCandidateConfiguration中获取，
  这个方法是用来获取候选的配置。
*/
List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes)

// 用来获取所有候选的配置，从getSpringFactoriesLoaderFactoryClass()中获取
// 断言中，提示为从META-INF/spring.factories中读取
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
                                                                         getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
// 标记了EnableAutoConfiguration的类
// @SpringBootApplication
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
	return EnableAutoConfiguration.class;
}
```

SpringBoot项目启动的时候，会先导入AutoConfigurationImportSelector，这个类会帮我们选择所有候选的配置，我们需要导入的配置都是SpringBoot帮我们写好的一个一个的配置类，那么这些配置类的位置，存在与META-INF/spring.factories文件中，通过这个文件，Spring可以找到这些配置类的位置，于是去加载其中的配置。

![img](https://pic3.zhimg.com/80/v2-ec490d9baecef2b0ff77af59598c8c12_720w.jpg)

spring.factories中存在那么多的配置，每次启动时都是把它们全量加载吗？

@ConditionalOnXXX:如果其中的条件都满足，该类才会生效。

- @ConditionalOnBean // 当给定的在bean存在时,则实例化当前Bean
- @ConditionalOnMissingBean // 当给定的在bean不存在时,则实例化当前Bean
- @ConditionalOnClass // 当给定的类名在类路径上存在，则实例化当前Bean
- @ConditionalOnMissingClass // 当给定的类名在类路径上不存在，则实例化当前Bean

所以在加载自动配置类的时候，并不是将spring.factories的配置全量加载进来，而是通过这个注解的判断，如果注解中的类都存在，才会进行加载。

所以就实现了：在pom.xml文件中加入stater启动器，SpringBoot自动进行配置。完成开箱即用。

**总结**

1. 整合过程中，通过导入starter启动器，Spring boot默认进行加载，使用者只需要写明一些配置即可使用，同时还进行了版本约束
2. 

如何自动装配？

使用spi机制去批量装配



SpringBoot所有自动配置类都是在启动的时候进行扫描并加载，通过spring.factories可以找到自动配置类的路径，但是不是所有存在于spring,factories中的配置都进行加载，而是通过@ConditionalOnClass注解进行判断条件是否成立（只要导入相应的stater，条件就能成立），如果条件成立则加载配置类，否则不加载该配置类。

在这里贴一个比较容易理解的过程：

- SpringBoot在启动的时候从类路径下的META-INF/spring.factories中获取EnableAutoConfiguration指定的值
- 将这些值作为自动配置类导入容器 ， 自动配置类就生效 ， 帮我们进行自动配置工作；
- 以前我们需要自己配置的东西 ， 自动配置类都帮我们解决了
- 整个J2EE的整体解决方案和自动配置都在springboot-autoconfigure的jar包中；
- 它将所有需要导入的组件以全类名的方式返回 ，这些组件就会被添加到容器中 ；
- 它会给容器中导入非常多的自动配置类 （xxxAutoConfiguration）, 就是给容器中导入这个场景需要的所有组件 ， 并配置好这些组件 ；
- 有了自动配置类 ， 免去了我们手动编写配置注入功能组件等的工作；

**参考**

https://zhuanlan.zhihu.com/p/95217578



## 4. SpringBoot启动流程

程序入口为

```java
@SpringBootApplication
public class Application {
  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}
```

分为两个部分

- 初始化方法

1. 推断应用的类型是普通项目还是web项目
2. 查找并加载所有可用初始化容器，设置到initializers属性中
3. 找出所有应用程序的监听器，设置到Listeners属性中
4. 推断并设置main方法定义类，找到运行的主类

构造器

```java
ublic SpringApplication(ResourceLoader resourceLoader, Class... primarySources) {
    // ......
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.setInitializers(this.getSpringFactoriesInstances();
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

- run方法流程

![img](https://img2020.cnblogs.com/i-beta/1418974/202003/1418974-20200309184347408-1065424525.png)



1. 实例对象run
2. 初始化监听器
   1. 创建应用监听器，开始监听
3. 读取环境配置参数
4. 打印banner







[狂神说Java](https://www.bilibili.com/video/BV1PE411i7CV?p=7)

https://www.cnblogs.com/hellokuangshen/p/12450327.html

## 5. SpringBoot中自定义启动器













## 6. SpringBoot中注解



















