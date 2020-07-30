# Spring Cloud

## 一 Spring Cloud架构原理

Spring Cloud核心组件

**初级系统可以使用**

**Eureka** 服务注册发现、心跳与故障

Eureka中服务注册信息主要保存在服务注册表、ReadWrite缓存和ReadOnly缓存中。服务注册时写入服务信息到注册表中，立马同步到ReadWrite缓存中，默认定时30s同步数据到ReadOnly缓存中。其它服务发现时，从ReadOnly缓存中定时拉取服务信息，默认30s拉取一次，读取不到再从ReadWrite缓存中拿。通过两级缓存，避免并发读写服务注册信息。

Eureka通过服务间心跳，来判断服务是否故障。服务定时30s发送心跳到Eureka Server中，线程定时60s会去检查服务注册表中服务信息。当超过90s后没有心跳，才认为出现故障。

**Feign** 服务调用，调用远程服务

对需要对外提供服务的接口，打上注解，会将该接口生成动态代理。调用feign动态代理后的接口时，会生成http协议格式的请求，完成请求地址URI的拼接。

**Ribbon** 负载均衡，负责选择从多个相同服务中进行选择

Ribbon会从本地的Eurkea注册表中获取提供该服务的机器列表，进行负载均衡，选择一台设备，再拼接请求地址的IP和端口，可以进行请求发送了。发送请求时，通信框架可以使用HttpClient组件。

**Zuul/Spring Cloud Gateway** 网关系统，配置转发路由接口

网关中会配置不同请求路径和服务的对应关系，请求到了网关，查找到提供服务的设备，将请求转发至对应设备。

**保证高可用**

**Hystrix** 限流熔断



## 二 Spring Cloud与Dubbo底层RPC调用对比？

Dubbo中，RPC性能比HTTP性能更好，并发能力更好。

Spring Cloud中RPC，采用HTTP请求，虽然慢点，但是也可以满足要求。

Spring Cloud全家桶组件更多，能够完成整套微服务架构搭建。Dubbo主要是服务框架，微服务架构下还需要配合其它组件使用。





