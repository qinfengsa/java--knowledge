## SpringBoot 

SpringBoot 框架是为了能够帮助使用spring 框架的开发者快速高效的构建一个基于Spring 框架以及Spring 生态体系的应用解决方案。它是对“约定优于配置”这个理念下的一个最佳实践。因此它是一个服务于框架的框架，服务的范围是简化配置文件。

## 约定优于配置





## SpringBoot 创造的新东西

SpringBoot没有新东西，都是基于已有的Spring体系构建的

1. AutoConfiguration 自动装配
2. Starter
3. Actuator 服务监控与管理 
4. SpringBoot CLI 命令控制行

## SpringBoot 启动流程



## SpringBootApplication

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    
}
~~~

SpringBootApplication 本质上是由3 个注解组成，分别是
1. @Configuration
2. @EnableAutoConfiguration
3. @ComponentScan

## SpringBoot Starter

Starter 是Spring Boot 中的一个非常重要的概念，Starter相当于模块，它能将模块所需的依赖整合起来并对模块内的Bean 根据环境（条件）进行自动配置。使用者只需要依赖相应功能的Starter，无需做过多的配置和依赖，SpringBoot 就能自动扫描并加载相应的模块。