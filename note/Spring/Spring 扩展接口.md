

## **Bean生命周期**

Spring的Bean实例化过程不是一启动容器就实例化的，而是调用BeanFactory 的 `getBean()` 方法时触发 Bean 的实例化进程；而且Bean的实例化是分阶段的，Spring在每个阶段为我们提供了相应的扩展接口，我们可以通过这些接口在Bean的生命周期内对Bean进行修改（功能增强，替换，篡改）

Bean的生命周期可以表达为：Bean的定义——Bean的实例化——Bean的初始化——Bean的使用——Bean的销毁 

1. Spring BeanDefinition配置阶段

2. Spring BeanDefinition解析阶段

3. Spring BeanDefinition注册阶段

4. Spring BeanDefinition合并阶段

5. BeanClass加载阶段(ClassLoading)

6. Bean实例化前阶段( BeforeInstantiation)

   BeanPostProcessor接口的postProcessBeforeInitialization

7. Bean实例化( Instantiation)

8. Bean实例化后阶段(AfterInstantiation)

   BeanPostProcessor接口的postProcessAfterInitialization

9. Spring Bean属性性赋值前阶段( Before Property Values Applied)

10. Aware接口回调阶段

    BeanNameAware ，BeanClassLoaderAware ，BeanFactoryAware 三大Aware接口

11. Bean初始化前阶段( BeforeInitialization)

    BeanPostProcessor接口的postProcessBeforeInitialization

12. Bean初始化( Initializing)

    InitializingBean 接口的afterPropertiesSet 方法， bean 配置的 init-method 方法

13. Bean初始化后( AfterInitialization)

    BeanPostProcessor接口的postProcessAfterInitialization

14. Spring Bean初始化完成阶段

15. Spring Bean销毁前( BeforeDestroy)

16. Bean销毁中( Destroying)

17. Bean销毁后( AfterDestroy)

18. Spring Bean垃圾收集(GC)

## Spring扩展接口分析

### 启动阶段

BeanFactoryPostProcessor接口，在容器启动阶段，允许我们在容器实例化 Bean 之前对注册到该容器的 BeanDefinition 做出修改

~~~java
// 提供扩展机制，允许在bean实例化之前，对bean的BeanDefinition做出修改 
public class BeanFactoryPostProcessorTest implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) 
        throws BeansException {
        // 获取指定的 BeanDefinition
        BeanDefinition bd = configurableListableBeanFactory.getBeanDefinition("student");
        MutablePropertyValues mpv = bd.getPropertyValues();
        // 修改 BeanDefinition
        mpv.addPropertyValue("name","bcd");
        mpv.addPropertyValue("age",11);
    }
}
~~~

一般情况下我们是不会主动去自定义 BeanFactoryPostProcessor ，其实 Spring 为我们提供了几个常用的BeanFactoryPostProcessor，他们是PropertyPlaceholderConfigurer 和 PropertyOverrideConfigurer ；

在AbstractApplicationContext类的refresh()方法中调用

~~~java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) { 
        try { 
            // ...
            // 实例化并调用所有注册的BeanFactoryPostProcessor接口的bean
            invokeBeanFactoryPostProcessors(beanFactory); 
           	// ...
        }  
    }
} 
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate
        .invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors()); 
}

~~~

最终会调用PostProcessorRegistrationDelegate的invokeBeanDefinitionRegistryPostProcessors方法和invokeBeanFactoryPostProcessors方法

#### **BeanDefinitionRegistryPostProcessor**

BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor的子接口，可以动态注册BeanDefinition到Spring容器，Spring的注解驱动实现就向容器注入了ConfigurationClassPostProcessor类，ConfigurationClassPostProcessor处理Config的相关注解

~~~java
private static void invokeBeanDefinitionRegistryPostProcessors(
    Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, 
    BeanDefinitionRegistry registry) {
	// 遍历BeanFactoryPostProcessor 接口
    for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessBeanDefinitionRegistry(registry);
    }
}
~~~

#### **BeanFactoryPostProcessor**

~~~java
private static void invokeBeanFactoryPostProcessors(
    Collection<? extends BeanFactoryPostProcessor> postProcessors,
    ConfigurableListableBeanFactory beanFactory) {
	// 遍历BeanFactoryPostProcessor 接口
    for (BeanFactoryPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessBeanFactory(beanFactory);
    }
}
~~~

### 实例化

~~~java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
   		try {
			// InstantiationAwareBeanPostProcessor接口
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}  
		try {
			Object beanInstance = doCreateBean(beanName, mbdToUse, args); 
			return beanInstance;
		}
    }
    
    // 真正创建Bean的方法
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, 
		final @Nullable Object[] args) throws BeanCreationException {
 
		// 用BeanWrapper封装被创建的Bean对象
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			// 创建bean实例
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		// 获取实例化对象的类型
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		} 
		// 调用 MergedBeanDefinitionPostProcessor 接口
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}  
				mbd.postProcessed = true;
			}
		} 
		// ...
		Object exposedObject = bean;
		try { 
            // 对 bean实例 进行填充，将各个属性值注入，可能存在依赖于其他 bean 的属性
            // 则会递归初始依赖 bean
			populateBean(beanName, mbd, instanceWrapper);
			// 初始化Bean对象
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		// ... 
 
		// 注册完成依赖注入的Bean
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		// catch ...  

		return exposedObject;
	}
}
~~~



#### InstantiationAwareBeanPostProcessor

InstantiationAwareBeanPostProcessor继承自BeanPostProcessor，InstantiationAwareBeanPostProcessor 调用时机是bean实例化（**Instantiation**）阶段，用于替换bean默认创建方式， 主要用于基础框架层面

~~~java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        // 遍历 InstantiationAwareBeanPostProcessor
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 调用前置方法
            Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
            if (result != null) {
                return result;
            }
        }
    }
    return null;
} 
~~~

#### MergedBeanDefinitionPostProcessor

主要处理合并后的BeanDefinition，其子类AutowiredAnnotationBeanPostProcessor提供了属性自动注入的功能

~~~java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, 
    String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        // 遍历 MergedBeanDefinitionPostProcessor
        if (bp instanceof MergedBeanDefinitionPostProcessor) {
            MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
            bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
        }
    }
}
~~~

**postProcessAfterInstantiation**

InstantiationAwareBeanPostProcessor的后置方法在populateBean中调用

~~~java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // ...
    // !mbd.isSynthetic() bean 不是程序生成的,不是AOP代理
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                // InstantiationAwareBeanPostProcessor 的后置方法,实例化之后,初始化之前
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 返回值为是否继续填充 bean 
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    } 
    // ...
}
~~~



### 初始化

~~~java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    // JDK的安全机制验证权限
    if (System.getSecurityManager() != null) { 
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            // aware接口方法调用
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }  else {
        // aware接口方法调用
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    // BeanPostProcessor 接口的前置处理 
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    } 
    // 初始化方法, InitializingBean接口和 init-method方法
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    } 
    // BeanPostProcessor 接口的后置处理 
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
~~~



#### **Aware接口**

Aware 接口是 Spring 容器的核心接口，实现了该接口的 bean 会通过事件回调机制完成Spring容器的通知功能；

在AbstractAutowireCapableBeanFactory中的initializeBean方法会调用Aware接口的方法

**部分 Aware 接口是通过BeanPostProcessor的实现类ApplicationContextAwareProcessor实现的**

~~~java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
    private void invokeAwareMethods(final String beanName, final Object bean) {
        if (bean instanceof Aware) {
            // 实现 BeanNameAware 接口，会执行 setBeanName
            if (bean instanceof BeanNameAware) { 
                ((BeanNameAware) bean).setBeanName(beanName);
            }
            // 实现 BeanClassLoaderAware 接口，会执行 setBeanClassLoader
            if (bean instanceof BeanClassLoaderAware) {
                ClassLoader bcl = getBeanClassLoader();
                if (bcl != null) {
                    ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
                }
            }
            // 实现 BeanFactoryAware 接口，会执行 setBeanFactory
            if (bean instanceof BeanFactoryAware) {
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
            }
        }
    }
}
~~~

#### **BeanPostProcessor接口**

在 Bean 实例化前后，我们可以通过自定义BeanPostProcessor拦截所有的bean（在bean实例化之前和之后拦截），对bean做增强处理（前、后置处理），相当于bean实例化前后插入了方法 

~~~java
public interface BeanPostProcessor { 
	// 为在Bean的初始化前提供回调入口
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	} 
	// 为在Bean的初始化之后提供回调入口
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	} 
} 
~~~

~~~java
// 原理 applyBeanPostProcessorsBeforeInitialization方法和applyBeanPostProcessorsAfterInitialization
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
	protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
 
		Object wrappedBean = bean;
		//对BeanPostProcessor后置处理器的postProcessBeforeInitialization
		//回调方法的调用，为Bean实例初始化前做一些处理
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		//调用Bean实例对象初始化的方法，这个初始化方法是在Spring Bean定义配置
		//文件中通过init-method属性指定的
		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		// 对BeanPostProcessor后置处理器的postProcessAfterInitialization
		// 回调方法的调用，为Bean实例初始化之后做一些处理
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		} 
		return wrappedBean;
	}
	//调用BeanPostProcessor后置处理器->实例化对象之前
	@Override 
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {
		Object result = existingBean;
		// 遍历 所有BeanPostProcessor 
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) { 
			// 在Bean实例初始化之前做一些自定义的处理操作
			Object current = beanProcessor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
	//调用BeanPostProcessor后置处理器->实例化对象之前
	@Override 
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException { 
		Object result = existingBean;
		//遍历 所有BeanPostProcessor 
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) { 
			// 在Bean实例初始化之后做一些自定义的处理操作
			Object current = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
}
~~~

##### ApplicationContextAwareProcessor

**部分 Aware 接口是通过BeanPostProcessor方法拦截实现的**

例如ApplicationContextAware接口就是在ApplicationContextAwareProcessor类中添加的

~~~java
class ApplicationContextAwareProcessor implements BeanPostProcessor {

	private final ConfigurableApplicationContext applicationContext;

	private final StringValueResolver embeddedValueResolver; 
 
	public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());
	}

	// Bean的初始化前执行
	@Override
	@Nullable
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
					bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
					bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                // 调用Aware 接口
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		} else {
            
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean)
                .setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
            // 实现 ApplicationContextAware 接口，会执行 setApplicationContext
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		return bean;
	}

}
~~~





#### **InitializingBean 和 init-method**

Spring 的 InitializingBean 接口为 bean 提供了定义初始化方法的方式，它仅包含了一个方法：`afterPropertiesSet()`。

~~~java
public interface InitializingBean { 
	void afterPropertiesSet() throws Exception; 
}
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
	// 在进行 “Aware 接口” 和 “BeanPostProcessor 前置处理”之后
	// 执行invokeInitMethods方法，判断bean有没有实现InitializingBean接口
	protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable { 
		// 判断bean有没有实现InitializingBean接口
		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || 
			!mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			// 存在安全机制，执行native方法
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			} else { // 否则直接执行
				((InitializingBean) bean).afterPropertiesSet();
			}
		}
		if (mbd != null && bean.getClass() != NullBean.class) {
			// 判断当前bean的init-method属性，通过反射调用
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}	
} 
~~~

