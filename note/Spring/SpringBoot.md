## SpringBoot 

SpringBoot 框架是为了能够帮助使用spring 框架的开发者快速高效的构建一个基于Spring 框架以及Spring 生态体系的应用解决方案。它是对“约定优于配置”这个理念下的一个最佳实践。因此它是一个服务于框架的框架，服务的范围是简化配置文件。



> 在SpringBoot2.0，如果应用采用SpringWebMVC作为web服务，默认情况下使用Tomcat容器
>
> 如果采用SpringWebFlux，默认情况下使用Web Netty Server

### 约定优于配置

约定优于配置（convention over configuration），也称作按约定编程，是一种软件设计范式，旨在减少软件开发人员需做决定的数量，获得简单的好处，而又不失灵活性。开发人员仅需规定应用中不符约定的部分。

如果所用工具的约定与预期相符，便可省去配置；反之，可以通过配置来达到预期的效果。

### SpringBoot 创造的新东西

SpringBoot没有新东西，都是基于已有的Spring体系构建的

1. AutoConfiguration 自动装配
2. Starter
3. Actuator 服务监控与管理 
4. SpringBoot CLI 命令控制行

## SpringBoot 启动流程

**启动代码**

~~~java
@SpringBootApplication 
public class SpringbootDemoApplication { 
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(SpringbootDemoApplication.class, args); 
    } 
}
~~~

SpringBoot 是基于`Spring注解驱动`实现的，掌握了注解驱动，很容易理解SpringBoot 的源码

~~~java
public class SpringApplication {
    // run 方法
    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
		return run(new Class<?>[] { primarySource }, args);
	} 
	public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        // 新建了一个SpringApplication对象
		return new SpringApplication(primarySources).run(args);
	}
}
~~~

### SpringApplication

~~~java
// 构造方法
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 这里是重点,判断容器类型 servlet or reactive 默认采用servlet
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 重点方法 getSpringFactoriesInstances 
    // 通过读取spring.factories文件 获取相应的类名称
    // 然后设置Initializers和Listeners
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
~~~

#### 容器类型

~~~java
static WebApplicationType deduceFromClasspath() {
    // 存在reactive相关类 springframework.web.reactive.DispatcherHandler
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) 
        // 不存在servlet相关类 web.servlet.DispatcherServlet
        && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
        // 不存在servlet容器 org.glassfish.jersey.servlet.ServletContainer
        && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        
        // 返回reactive 
        return WebApplicationType.REACTIVE;
    }
    // "javax.servlet.Servlet","org.springframework.web.context.ConfigurableWebApplicationContext"
    // 这两个类不存在,返回none
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    // 返回servlet
    return WebApplicationType.SERVLET;
}
~~~

#### getSpringFactoriesInstances

~~~~java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
} 
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    // 读取spring.factories文件 获取相应的类名称集合
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 通过反射创建无参构造的实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
} 
~~~~

##### SpringFactoriesLoader

~~~java
public final class SpringFactoriesLoader {
	// 配置文件位置和名称
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
    // 类加载隔离,不同的ClassLoader 对应不同的 MultiValueMap<String, String>
    // MultiValueMap<K, V> extends Map<K, List<V>> 最后返回的结果是List
	private static final Map<ClassLoader, MultiValueMap<String, String>> cache 
        = new ConcurrentReferenceHashMap<>();
    // 最后返回的结果是List
    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
        // 通过key=classLoader获取对应的MultiValueMap
        // 然后从MultiValueMap中key=factoryClassName获取需要加载的类名称集合
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}

	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        // 从缓存中获取当前类加载器的 MultiValueMap
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
            // 把META-INF/spring.factories文件解析为URL集合
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
                // 遍历获取 properties 
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
                    // 加入MultiValueMap集合 
					String factoryClassName = ((String) entry.getKey()).trim();
					for (String factoryName : StringUtils
                         .commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryClassName, factoryName.trim());
					}
				}
			}
            // 放入缓存
			cache.put(classLoader, result);
			return result;
		} // catch ...
	}
}
~~~

##### spring.factories

~~~properties
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer
~~~

### 启动

~~~java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    // 调用getSpringFactoriesInstances方法 
    // 读取spring.factories文件 获取 SpringApplicationRunListener接口对应的类名
    // 然后创建监听实例
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 监听器,触发starting事件
    listeners.starting();
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        // 创建容器
        context = createApplicationContext();
        // 读取spring.factories文件 获取 SpringBootExceptionReporter接口对应的类名,异常时调用
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
             new Class[] { ConfigurableApplicationContext.class }, context);
        // 准备
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 刷新context
        refreshContext(context);
        // afterRefresh 空方法,扩展使用
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // 监听器,触发 started 事件
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        // 处理异常报告
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 监听器,触发 running 事件
        listeners.running(context);
    }
    catch (Throwable ex) {
        // 处理异常报告
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
~~~

#### 创建容器

~~~java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            // 根据前面的构造方法的容器类型;选择相应的class
            switch (this.webApplicationType) {
                case SERVLET:
                    // servlet容器 AnnotationConfigServletWebServerApplicationContext
                    contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                    break;
                case REACTIVE:
                    // reactive容器 AnnotationConfigReactiveWebServerApplicationContext
                    contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                    break;
                default:
                    // 默认 AnnotationConfigApplicationContext
                    contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        } // catch ...
         
    }
    // 反射调用无参构造方法实例化
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
~~~

##### **构造方法**

可以看到，三个class的无参构造是没有区别的，熟悉注解驱动的话应该知道：AnnotatedBeanDefinitionReader的构造方法中注册了注解驱动相关的bean，最重要的两个是ConfigurationClassPostProcessor和AutowiredAnnotationBeanPostProcessor

**ConfigurationClassPostProcessor**：负责解析加了@Configuration的配置类，还会解析@ComponentScan、@ComponentScans注解扫描的包，以及@Import等注解

**AutowiredAnnotationBeanPostProcessor**：@Autowired 注解扫描 自动注入

~~~java
public AnnotationConfigServletWebServerApplicationContext() {
    // 带注解的BeanDefinition的读取器 
    this.reader = new AnnotatedBeanDefinitionReader(this);
    // 按类路径扫描的BeanDefinition扫描器
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
public AnnotationConfigReactiveWebServerApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
public AnnotationConfigApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
~~~

#### prepareContext

~~~java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
  SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    // 设置环境
    context.setEnvironment(environment);
    // 注册beanName生成器
    // 设置容器的ResourceLoader和ClassLoader
    // 设置ApplicationConversionService 转换服务
    postProcessApplicationContext(context);
    // 调用构造方法 生产的ApplicationContextInitializer接口实例的方法
    applyInitializers(context);
    // 监听器,触发 contextPrepared 事件
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // Load the sources
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 创建BeanDefinitionLoader
    load(context, sources.toArray(new Object[0]));
    // 监听器,触发 contextLoaded 事件
    listeners.contextLoaded(context);
}
~~~

##### applyInitializers

~~~java
protected void applyInitializers(ConfigurableApplicationContext context) {
    // 遍历构造方法中生产的ApplicationContextInitializer接口实例(从spring.factories文件中获取)
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver
            .resolveTypeArgument(initializer.getClass(),ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        // 调用ApplicationContextInitializer接口的方法
        initializer.initialize(context);
    }
}
~~~

#### refreshContext

调用AbstractApplicationContext的refresh()方法，IOC的核心流程

~~~java
private void refreshContext(ConfigurableApplicationContext context) {
    refresh(context);
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}
protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    ((AbstractApplicationContext) applicationContext).refresh();
}
~~~

**refresh**

~~~java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 准备方法: 设置启动时间,启动标志,关闭标志
        prepareRefresh();
        // 通知子类执行refreshBeanFactory()方法,创建BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // 为BeanFactory配置容器特性，例如类加载器、事件处理器等
        prepareBeanFactory(beanFactory);
        try {
            // 子类在所有的bean尚未初始化之前注册BeanPostProcessor,默认空实现
            postProcessBeanFactory(beanFactory);
            // 实例化并调用所有注册的BeanFactoryPostProcessor接口的bean
            invokeBeanFactoryPostProcessors(beanFactory);
            // 注册拦截Bean创建的Bean处理器,即注册 BeanPostProcessor
            registerBeanPostProcessors(beanFactory);
            // 初始化信息源，和国际化相关.
            initMessageSource();
            // 初始化容器事件传播器.
            initApplicationEventMulticaster();
            // 给子类扩展初始化其他Bean
            onRefresh();
            // 在所有bean中查找listener bean，然后注册到广播器中
            registerListeners();
            // 初始化剩下的单例Bean(非延迟加载的)
            finishBeanFactoryInitialization(beanFactory);
            // 完成刷新过程,通知生命周期处理器lifecycleProcessor刷新过程,同时发出ContextRefreshEvent通知
            finishRefresh();
        } catch (BeansException ex) { 
            // 销毁已经创建的Bean
            destroyBeans(); 
            // 重置容器激活标签
            cancelRefresh(ex); 
            throw ex;
        }   finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
~~~

##### prepareRefresh

重写了prepareRefresh，清除扫描器的缓存

~~~java
public class AnnotationConfigServletWebServerApplicationContext extends ServletWebServerApplicationContext
		implements AnnotationConfigRegistry {	
	@Override
	protected void prepareRefresh() {
        // 清除扫描器的缓存
		this.scanner.clearCache();
		super.prepareRefresh();
	}
}
public class AnnotationConfigReactiveWebServerApplicationContext extends ReactiveWebServerApplicationContext
		implements AnnotationConfigRegistry {
	@Override
	protected void prepareRefresh() {
		this.scanner.clearCache();
		super.prepareRefresh();
	}
}
~~~





##### postProcessBeanFactory

子类在所有的bean尚未初始化之前注册BeanPostProcessor，默认空实现

~~~java
public class AnnotationConfigServletWebServerApplicationContext extends ServletWebServerApplicationContext
		implements AnnotationConfigRegistry {
	@Override
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 调用父类方法
		super.postProcessBeanFactory(beanFactory);
        // 默认启动方式 basePackages和annotatedClasses是空的
		if (!ObjectUtils.isEmpty(this.basePackages)) {
			this.scanner.scan(this.basePackages);
		}
		if (!this.annotatedClasses.isEmpty()) {
			this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
		}
	}
}
// 父类
public class ServletWebServerApplicationContext extends GenericWebApplicationContext
		implements ConfigurableWebServerApplicationContext {
	@Override
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 添加一个BeanPostProcessor接口类
		beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
        // 忽略接口ServletContextAware
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);
        // 注册特定于web的作用域 ("request", "session", "globalSession")
		registerWebApplicationScopes();
	}
}
~~~



##### onRefresh

在Spring IOC流程中，onRefresh是个空方法，AnnotationConfigServletWebServerApplicationContext 和AnnotationConfigReactiveWebServerApplicationContext的父类对其进行了重写，核心目的：创建web容器

~~~java
// AnnotationConfigServletWebServerApplicationContext 的父类
public class GenericWebApplicationContext extends GenericApplicationContext
		implements ConfigurableWebApplicationContext, ThemeSource {
	@Override
	protected void onRefresh() {
        // 初始化主题
		this.themeSource = UiApplicationContextUtils.initThemeSource(this);
	}
}
// AnnotationConfigServletWebServerApplicationContext 的父类
public class ServletWebServerApplicationContext extends GenericWebApplicationContext
		implements ConfigurableWebServerApplicationContext {	
	@Override
	protected void onRefresh() {
		super.onRefresh();
		try {
            // 创建web容器
			createWebServer();
		} //..
	}
    // 创建Servlet容器
    private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			ServletWebServerFactory factory = getWebServerFactory();
			this.webServer = factory.getWebServer(getSelfInitializer());
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context", ex);
			}
		}
		initPropertySources();
	}
}
// AnnotationConfigReactiveWebServerApplicationContext 的父类
public class ReactiveWebServerApplicationContext extends GenericReactiveWebApplicationContext
		implements ConfigurableWebServerApplicationContext {
	@Override
	protected void onRefresh() {
		super.onRefresh();
		try {
            // 创建web容器
			createWebServer();
		} //..
	}
    // 创建Reactive web 容器
    private void createWebServer() {
		ServerManager serverManager = this.serverManager;
		if (serverManager == null) {
			this.serverManager = ServerManager.get(getWebServerFactory());
		}
		initPropertySources();
	}
}
~~~

##### finishRefresh

重写了该方法，启动web容器，发布容器初始化事件

~~~java
public class ServletWebServerApplicationContext extends GenericWebApplicationContext
		implements ConfigurableWebServerApplicationContext {
	@Override
	protected void finishRefresh() {
		super.finishRefresh();
        // 启动web容器
		WebServer webServer = startWebServer();
		if (webServer != null) {
            // 发布容器初始化事件
			publishEvent(new ServletWebServerInitializedEvent(webServer, this));
		}
	}
}
public class ReactiveWebServerApplicationContext extends GenericReactiveWebApplicationContext
		implements ConfigurableWebServerApplicationContext {
	@Override
	protected void finishRefresh() {
		super.finishRefresh();
        // 启动reactive容器
		WebServer webServer = startReactiveWebServer();
		if (webServer != null) {
            // 发布容器初始化事件
			publishEvent(new ReactiveWebServerInitializedEvent(webServer, this));
		}
	}
}
~~~

## SpringBootApplication

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration // 本质是一个@Configuration,声明是一个配置类文件
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    
}
~~~

SpringBootApplication 本质上是由3 个注解组成，分别是
1. @Configuration  声明当前类是一个配置类
2. @EnableAutoConfiguration 激活自动装配
3. @ComponentScan 扫描目录下的bean

### **EnableAutoConfiguration**

我们可以@EnableAutoConfiguration实际相当于import了两个类；

@Import注解可以导入3种类型的class

* ImportSelector 接口：调用selectImports方法
* ImportBeanDefinitionRegistrar 接口：调用registerBeanDefinitions方法，注册BeanDefinition
* 其他配置类（需要防止循环导入）：递归进行解析

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
 
	Class<?>[] exclude() default {}; 
	String[] excludeName() default {}; 
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

} 
~~~

 

### **AutoConfigurationImportSelector**

实现了DeferredImportSelector接口，这个接口有单独的处理逻辑， 把配置文件中的class加载到容器中，最后处理

为什么要最后处理，例如：ConditionalOnBean条件在SPI阶段过滤时，有可能对应的bean还没有注入到容器，所以在SPI阶段只需要判断对应的class是否存在即可；条件过滤（注解过滤）就需要需要等前面的配置类加载完后，最后处理

~~~java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
    // 内部类        
	private static class AutoConfigurationGroup
	implements DeferredImportSelector.Group, BeanClassLoaderAware, BeanFactoryAware, ResourceLoaderAware {
		// 配置class 集合
		private final Map<String, AnnotationMetadata> entries = new LinkedHashMap<>();

		private final List<AutoConfigurationEntry> autoConfigurationEntries = new ArrayList<>();
        
        @Override
		public void process(AnnotationMetadata annotationMetadata, 
                            DeferredImportSelector deferredImportSelector) {
			Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
					() -> String.format("Only %s implementations are supported, got %s",
							AutoConfigurationImportSelector.class.getSimpleName(),
							deferredImportSelector.getClass().getName()));
            // 1.getAutoConfigurationMetadata()
            // 2.getAutoConfigurationEntry()
			AutoConfigurationEntry autoConfigurationEntry = 
                ((AutoConfigurationImportSelector) deferredImportSelector)
					.getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
			this.autoConfigurationEntries.add(autoConfigurationEntry);
            // 最终把解析的配置class 放入 entries 集合
			for (String importClassName : autoConfigurationEntry.getConfigurations()) {
				this.entries.putIfAbsent(importClassName, annotationMetadata);
			}
		}
    }
}            
~~~

 

#### getAutoConfigurationMetadata

从spring-autoconfigure-metadata.properties文件中获取配置信息

~~~java
private AutoConfigurationMetadata getAutoConfigurationMetadata() {
    if (this.autoConfigurationMetadata == null) {
        this.autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
    }
    return this.autoConfigurationMetadata;
}

final class AutoConfigurationMetadataLoader {
	// 文件名称
	protected static final String PATH = "META-INF/spring-autoconfigure-metadata.properties";

	static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader) {
		return loadMetadata(classLoader, PATH);
	}

	static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader, String path) {
		try {
            // 把META-INF/spring-autoconfigure-metadata.properties文件解析为URL集合
			Enumeration<URL> urls = (classLoader != null) ? classLoader.getResources(path)
					: ClassLoader.getSystemResources(path);
			Properties properties = new Properties();
             // 遍历获取 properties 
			while (urls.hasMoreElements()) {
				properties.putAll(PropertiesLoaderUtils.loadProperties(new UrlResource(urls.nextElement())));
			}
			return loadMetadata(properties);
		} // catch ...
	}
    // 创建一个PropertiesAutoConfigurationMetadata,封装了properties属性
    static AutoConfigurationMetadata loadMetadata(Properties properties) {
		return new PropertiesAutoConfigurationMetadata(properties);
	}
}
~~~

**spring-autoconfigure-metadata.properties**

~~~properties
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration.ConditionalOnWebApplication=SERVLET
org.springframework.boot.autoconfigure.web.reactive.ReactiveWebServerFactoryAutoConfiguration.ConditionalOnWebApplication=REACTIVE
#...省略
~~~

#### getAutoConfigurationEntry

这里的annotationMetadata是@SpringBootApplication注解

~~~java
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata, AnnotationMetadata annotationMetadata) {
    // autoConfigurationMetadata 是前面获取到的
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 获取配置文件中的class 列表
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // 去重
    configurations = removeDuplicates(configurations);
    // 获取需要排除的集合 @SpringBootApplication 的exclude和excludeName
    // (只能排除auto-configuration classes,否则会报错)
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    // 移除需要排除的class
    configurations.removeAll(exclusions);
    // 过滤
    configurations = filter(configurations, autoConfigurationMetadata);
    // 触发事件 AutoConfigurationImportEvent
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
~~~

##### SpringFactoriesLoader

可以看出这里调用了SpringFactoriesLoader，需要解析`META-INF/spring.factories`文件中的EnableAutoConfiguration类信息；这里只是拿到了类的name，没有实例化

~~~java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) { 
    // 只是拿到了类的name,没有实例化
    List<String> configurations = SpringFactoriesLoader
        .loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),getBeanClassLoader());
    Assert.notEmpty(configurations, "");
    return configurations;
}
// key = org.springframework.boot.autoconfigure.EnableAutoConfiguration
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}
~~~

**EnableAutoConfiguration配置信息**

这些类全部是配置类（@Configuration注解），需要逐一导入，但是并不需要全部注册，所以需要过滤（@Conditional注解）

~~~properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
#...省略
~~~

##### 过滤

这里的configurations集合中的class还没有实例化

~~~java
private List<String> filter(List<String> configurations,AutoConfigurationMetadata autoConfigurationMetadata){
    long startTime = System.nanoTime();
    // AutoConfiguration类的name集合
    String[] candidates = StringUtils.toStringArray(configurations);
    boolean[] skip = new boolean[candidates.length];
    boolean skipped = false;
    // 获取过滤使用的class,并实例化
    for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
        // 如果 filter 实现了 BeanClassLoaderAware BeanFactoryAware  EnvironmentAware ResourceLoaderAware
        // 注入对应的信息
        invokeAwareMethods(filter);
        // 使用AutoConfiguration类的name集合 匹配,获取到匹配结果
        boolean[] match = filter.match(candidates, autoConfigurationMetadata);
        for (int i = 0; i < match.length; i++) {
            if (!match[i]) {
                // 移除 
                skip[i] = true; 
                candidates[i] = null;
                skipped = true;
            }
        }
    }
    if (!skipped) {
        return configurations;
    }
    // 把结果放入新的集合,过滤掉不符合条件的class
    List<String> result = new ArrayList<>(candidates.length);
    for (int i = 0; i < candidates.length; i++) {
        if (!skip[i]) {
            result.add(candidates[i]);
        }
    } 
    return new ArrayList<>(result);
}
// 获取META-INF/spring.factories文件中的AutoConfigurationImportFilter类信息
protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
    return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
}
~~~

**spring.factories**

~~~properties
# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition
~~~

###### FilteringSpringBootCondition

FilteringSpringBootCondition有3个子类OnBeanCondition，OnClassCondition，OnWebApplicationCondition

~~~java
abstract class FilteringSpringBootCondition extends SpringBootCondition
		implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {
    
    @Override
	public boolean[] match(String[] autoConfigurationClasses, 
                           AutoConfigurationMetadata autoConfigurationMetadata) {
		ConditionEvaluationReport report = ConditionEvaluationReport.find(this.beanFactory);
		ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses, autoConfigurationMetadata);
        // 匹配数组
		boolean[] match = new boolean[outcomes.length];
		for (int i = 0; i < outcomes.length; i++) {
            // 判断是否匹配
			match[i] = (outcomes[i] == null || outcomes[i].isMatch());
			if (!match[i] && outcomes[i] != null) {
				 
				if (report != null) {
					report.recordConditionEvaluation(autoConfigurationClasses[i], this, outcomes[i]);
				}
			}
		}
		return match;
	}
    // 匹配结果,抽象方法,交给子类
    protected abstract ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
			AutoConfigurationMetadata autoConfigurationMetadata);
}
~~~

**OnBeanCondition**

例：class = org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration

~~~properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
~~~

1.在spring.factories文件找到`key = EnableAutoConfiguration`的其中一个`value = CacheAutoConfiguration`

2.组合`key = CacheAutoConfiguration.ConditionalOnBean`，然后去spring-autoconfigure-metadata.properties文件查询这个key对应的`value = CacheAspectSupport`

~~~properties
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration.ConditionalOnBean=org.springframework.cache.interceptor.CacheAspectSupport
~~~

3.使用classLoader加载得到的class:`CacheAspectSupport`，加载成功则把配置类`CacheAutoConfiguration`导入容器中，然后解析配置类

**OnClassCondition**

例：class = org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration

~~~properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
~~~

1.在spring.factories文件找到`key = EnableAutoConfiguration`的其中一个`value = WebMvcAutoConfiguration`

2.组合`key = WebMvcAutoConfiguration.ConditionalOnClass`，然后去spring-autoconfigure-metadata.properties文件查询这个key对应的`value = DispatcherServlet`

~~~properties
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.ConditionalOnClass=javax.servlet.Servlet,org.springframework.web.servlet.config.annotation.WebMvcConfigurer,org.springframework.web.servlet.DispatcherServlet
~~~

3.使用classLoader加载得到的class:`DispatcherServlet`，加载成功则把配置类`WebMvcAutoConfiguration`导入容器中，然后解析配置类

**OnWebApplicationCondition**

value只有3个SERVLET，REACTIVE和空

例：class = org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration

~~~properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
~~~

1.在spring.factories文件找到`key = EnableAutoConfiguration`的其中一个`value = WebMvcAutoConfiguration`

2.组合`key = WebMvcAutoConfiguration.ConditionalOnWebApplication`，然后去spring-autoconfigure-metadata.properties文件查询这个key对应的`value = SERVLET`

~~~properties
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.ConditionalOnWebApplication=SERVLET
~~~

3.结果是SERVLET，需要去加载类GenericWebApplicationContext，加载成功则把配置类`WebMvcAutoConfiguration`导入容器中，然后解析配置类

~~~java
class OnWebApplicationCondition extends FilteringSpringBootCondition {
	// SERVLET 需要加载的类
	private static final String SERVLET_WEB_APPLICATION_CLASS = 
        "org.springframework.web.context.support.GenericWebApplicationContext";
	// REACTIVE 需要加载的类
	private static final String REACTIVE_WEB_APPLICATION_CLASS = 
        "org.springframework.web.reactive.HandlerResult";
}
~~~



##### processImports

第一层过滤（SPI）完成后，会回调`ConfigurationClassParser.processImports()`方法，解析导入的配置类，@Configuration注解的类会调用processConfigurationClass方法，在这里会扫描@Conditional注解，按条件进行装配(第二层过滤)





### **AutoConfigurationPackages.Registrar**

实现了ImportBeanDefinitionRegistrar接口，调用registerBeanDefinitions方法

~~~java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        register(registry, new PackageImport(metadata).getPackageName());
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.singleton(new PackageImport(metadata));
    }

}
~~~



主要的用途：List<String> packages = AutoConfigurationPackages.get(this.beanFactory);

第三方开发的AutoConfiguration（MybatisAutoConfiguration）可以通过这个方法获取到当前扫描的包名集合，去对应的包名下扫描自己定义的注解和配置，符合SpringBoot的规范

 

## SpringBoot Starter

Starter 是Spring Boot 中的一个非常重要的概念，Starter相当于模块，它能将模块所需的依赖整合起来并对模块内的Bean 根据环境（条件）进行自动配置。使用者只需要依赖相应功能的Starter，无需做过多的配置和依赖，SpringBoot 就能自动扫描并加载相应的模块。

在Spring框架体系下，我们如果需要引入其他模块（MyBatis，Dubbo），需要在XML配置文件中配置相关的bean，这些bean交给Spring容器管理，从而实现其他模块的功能

在SpringBoot框架体系下，由于自动配置的理念，引入了Starter，简单来说，原来需要通过XML文件<bean>标签引入的bean，我们通过@Configuration配置类（一般名称是xxAutoConfiguration）的@Bean注解来实现，最终实现的效果和XML配置是相同的

为了将我们扩展的xxAutoConfiguration导入Spring容器，SpringBoot提供了SPI机制

* 首先在`META-INF/spring.factories`文件中定义EnableAutoConfiguration属性的值(可以设多个)

  ~~~properties
  # Auto Configure
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=xxAutoConfiguration 
  ~~~

* 然后SpringBoot在启动阶段会扫描这个文件，获取到xxAutoConfiguration

* 之后存在一个过滤机制，SpringBoot会扫描`META-INF/spring-autoconfigure-metadata.properties`文件中的属性，实现条件装配

  ConditionalOnBean，ConditionalOnClass，OnWebApplicationCondition会影响配置类是否加载

  ~~~properties
  xxAutoConfiguration.ConditionalOnBean=# 存在bean导入,否则跳过
  xxAutoConfiguration.ConditionalOnClass=# 存在class导入,否则跳过
  xxAutoConfiguration.OnWebApplicationCondition=# 只有3个值SERVLET，REACTIVE和空
  ~~~

  AutoConfigureBefore，AutoConfigureAfter，AutoConfigureOrder会影响配置类加载的顺序

  ~~~properties
  xxAutoConfiguration.AutoConfigureBefore=# 在某些class之前加载
  xxAutoConfiguration.AutoConfigureAfter=# 在某些class之后加载
  xxAutoConfiguration.AutoConfigureOrder=# 序号
  ~~~

  







## SpringBoot 异常处理

 我们只需要在类上加上`@ControllerAdvice`注解，这个类就成为了全局异常处理类 

~~~java
/**
 * 全局异常 拦截处理 
 */
// ControllerAdvice 注解 可以针对特定的包或class生效
@ControllerAdvice(assignableTypes = {ExceptionController.class})
public class GlobalExceptionHandler {

    /**
     * 拦截所有异常Exception 
     */
    @ExceptionHandler(Exception.class)
    public String allException(Exception e) {
        // 返回错误页面
        return "/error/404";
    }

    /**
     * 拦截异常MyTestException,即使有拦截全部异常,也会单独生效 
     */
    @ExceptionHandler(MyTestException.class)
    @ResponseBody
    public Map<String, Object> myTestException(MyTestException e) {
        Map<String, Object> map = new HashMap<>();
        map.put("message", e.getMessage());
        // 返回错误文本信息
        return map;
    } 
} 
~~~





## SpringBoot Rest

Restful:

简化并统一api接口

 



