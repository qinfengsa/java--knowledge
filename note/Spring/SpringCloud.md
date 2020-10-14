## 分布式配置

Spring Cloud Config是 Spring Cloud团队创建的一个全新项目，用来为分布式系统中的基础设施和微服务应用提供集中化的外部配置支持，它分为服务端与客户端两个部分

* **服务端**：也称为分布式配置中心，是一个独立的微服务应用，用来连接配置仓库并为客户端提供获取配置信息、加密/解密信息等访问接口
* **客户端**：微服务架构中的其它应用，在启动时从配置中心获取和加载配置信息，实现配置的统一管理

**SpringBoot 配置的加载顺序**

1. 命令行输入的参数 
2. 



SpringCloud Bootstrap上下文

创建一个 Spring Cloud配置服务器，演示两种不同的机制来提供应用程序配置数据，一种使用文件系统，另一种使用Git存储库.



## 服务注册和发现

### Eureka

Spring Cloud Eureka是 Spring Cloud Netflix微服务套件中的一部分，它基于Netflix Eureka做了二次封装,主要负责完成微服务架构中的服务治理功能



~~~properties
### 取消服务器自我注册
eureka.client.register-with-eureka = false
### 注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = false
## Eureka Server 服务 URL,用于客户端注册
eureka.client.serviceUrl.defaultZone=\
  http://localhost:9090/eureka,http://localhost:9091/eureka
~~~



### 源码分析

首先启动EurekaServer需要添加`@EnableEurekaServer`注解，Spring注解驱动会解析@Import导入的类型

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

} 
~~~

#### EurekaServerMarkerConfiguration

这个类型是Configuration配置类，创建了一个bean：EurekaServerMarkerConfiguration.Marker；这里会让人看了一头雾水，为什么要创建一个空的对象，对象为空，那么这个Marker不是用来使用的，而是作为激活EurekaServer的条件，到Spring存在这个bean，EurekaServer相关的所有bean都会被自动装配

~~~java
@Configuration(proxyBeanMethods = false)
public class EurekaServerMarkerConfiguration {

	@Bean
	public Marker eurekaServerMarkerBean() {
		return new Marker();
	} 
	class Marker {

	} 
}
~~~

EurekaController





RestTemplate



Eureka Client负责下面的任务:

向 Eureka Server注册服务实例

向 Eureka Server服务租约

当服务关闭期间,向 Eureka Server取消租约

查询 Eureka Server中的服务实例列表



Eureka Client还需要配置一个 Eureka Server的URL列表.

## 负载均衡



## 服务熔断



## 服务调用



## 服务网关Gateway





## 消息总线Bus



## 分布式消息Stream





## Security