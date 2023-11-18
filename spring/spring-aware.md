# aware

## aware接口是干嘛用的？

框架具备高度封装性，我们接触到的一般都是业务代码，一个底层功能API不能轻易的获取到， 但是这不意味着永远用不到这些对象，如果用到了就可以使用框架提供的类似Aware的接口，让框架给我们注入该对象。可以让我们在自定义对象中获取容器对象。

在开发中，不可避免需要使用到Spring容器本身的功能和资源。这时，Bean需要意识到Spring容器的存在，才能使用Spring提供的资源，这就是所谓的Spring Aware。

## spring常用的aware接口有哪些？

| Aware接口               | 回调方法                                                     | 作用                                                       |
| ----------------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| ServletContextAware     | setServletContext(ServletContext context)                    | Spring框架回调方法注入ServletContext对象， web环境下才生效 |
| BeanFactoryAware        | setBeanFactory(BeanFactory factory)                          | Spring框架回调方法注入beanFactory对象                      |
| BeanNameAware           | setBeanName(String beanName)                                 | Spring框架回调方法注入当前Bean在容器中的beanName           |
| ApplicationContextAware | setApplicationContext(ApplicationContext applicat tionContext) | Spring框架回调方法注入applicationContext对象               |

## aware怎么用？

只需要实现相关的aware接口就可以，具体请看下面的例子。

### ApplicationContextAware

```java
public class MyBean implements ApplicationContextAware {
    private ApplicationContext applicationContext;
 
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
 
    public void doSomething() {
        // 获取其他的Bean实例或其他的组件
        OtherBean otherBean = applicationContext.getBean("otherBean", OtherBean.class);
        // ...
    }
}
```

### BeanFactoryAware

- 获取Spring容器中的Bean实例
- 手动注册BeanDefinition

```java
public class MyBean implements BeanFactoryAware {
    private BeanFactory beanFactory;
 
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
 
    public void doSomething() {
        // 获取其他的Bean实例或其他的组件
        OtherBean otherBean = beanFactory.getBean("otherBean", OtherBean.class);
        // ...
    }
}
```

### MessageSourceAware

- 获取国际化信息

```java
public class MyBean implements MessageSourceAware {
    private MessageSource messageSource;
 
    @Override
    public void setMessageSource(MessageSource messageSource) {
        this.messageSource = messageSource;
    }
 
    public void doSomething() {
        // 获取国际化信息
        String message = messageSource.getMessage("key", null, Locale.getDefault());
        // ...
    }
}
```

### ResourceLoaderAware

- 加载配置文件
- 加载图片等静态资源

```java
public class MyBean implements ResourceLoaderAware {
    private ResourceLoader resourceLoader;
 
    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }
 
    public void doSomething() {
        // 加载配置文件
        Resource resource = resourceLoader.getResource("classpath:config.xml");
        // ...
    }
}
```

### ServletContextAware

- 获取Web项目的一些信息，如上下文路径等

```java
public class MyBean implements ServletContextAware {
    private ServletContext servletContext;
 
    @Override
    public void setServletContext(ServletContext servletContext) {
        this.servletContext = servletContext;
    }
 
    public void doSomething() {
        // 获取上下文路径
        String contextPath = servletContext.getContextPath();
        // ...
    }
}
```

### EnvironmentAware

- 获取当前的环境配置，如开发环境、测试环境或生产环境
- 获取配置文件中的属性值

```java
public class MyBean implements EnvironmentAware {
    private Environment environment;
 
    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
 
    public void doSomething() {
        // 获取当前环境
        String profile = environment.getActiveProfiles()[0];
        // 获取配置文件中的属性值
        String value = environment.getProperty("key");
        // ...
    }
}
```

### ServletConfigAware

- 获取Servlet的名称
- 获取Servlet的初始化参数

```java
public class MyServlet extends HttpServlet implements ServletConfigAware {
    private ServletConfig servletConfig;
 
    @Override
    public void setServletConfig(ServletConfig servletConfig) {
        this.servletConfig = servletConfig;
    }
 
    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        // 获取Servlet的名称
        String servletName = servletConfig.getServletName();
        // 获取Servlet的初始化参数
        String value = servletConfig.getInitParameter("key");
        // ...
    }
}
```

### ApplicationContextInitializer

- 修改配置信息
- 注册BeanDefinition

```java
public class MyApplicationContextInitializer implements ApplicationContextInitializer {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        // 注册BeanDefinition
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(OtherBean.class);
        beanDefinition.setPropertyValues(new MutablePropertyValues());
        ((DefaultListableBeanFactory) applicationContext.getBeanFactory()).registerBeanDefinition("otherBean", beanDefinition);
        // ...
    }
}
```

### EmbeddedValueResolverAware

- 替换配置文件中的占位符

```java
public class MyBean implements EmbeddedValueResolverAware {
    private EmbeddedValueResolver embeddedValueResolver;
 
    @Override
    public void setEmbeddedValueResolver(EmbeddedValueResolver embeddedValueResolver) {
        this.embeddedValueResolver = embeddedValueResolver;
    }
 
    public void doSomething() {
        // 获取属性值
        String value = embeddedValueResolver.resolveStringValue("${key}");
        // ...
    }
}
```

### LoadTimeWeaverAware

- 动态加载类 

```java
public class MyBean implements LoadTimeWeaverAware {
    private LoadTimeWeaver loadTimeWeaver;
 
    @Override
    public void setLoadTimeWeaver(LoadTimeWeaver loadTimeWeaver) {
        this.loadTimeWeaver = loadTimeWeaver;
    }
 
    public void doSomething() {
        // 动态加载类
        ClassLoader classLoader = loadTimeWeaver.getClassLoader();
        Class&lt;?&gt; clazz = classLoader.loadClass("com.example.OtherClass");
        // ...
    }
}
```

### ApplicationEventPublisherAware

- 实现自定义事件
- 监听Spring容器事件

```java
public class MyBean implements ApplicationEventPublisherAware {
    private ApplicationEventPublisher applicationEventPublisher;
 
    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }
 
    public void doSomething() {
        // 发布事件
        applicationEventPublisher.publishEvent(new MyEvent(this, "event data"));
        // ...
    }
}
 
@Component
public class MyEventListener implements ApplicationListener&lt;MyEvent&gt; {
    @Override
    public void onApplicationEvent(MyEvent event) {
        // 处理事件
        // ...
    }
}
```

### ConversionServiceAware

- 类型转换
- 数据校验

```java
public class MyBean implements ConversionServiceAware {
    private ConversionService conversionService;
 
    @Override
    public void setConversionService(ConversionService conversionService) {
        this.conversionService = conversionService;
    }
 
    public void doSomething() {
        // 类型转换
        String str = "123";
        Integer value = conversionService.convert(str, Integer.class);
        // ...
    }
}
```

## aware底层是如何实现的？

在Spring中，类似于ApplicationContextAware接口的设计有很多，本质上，Spring中类似XxxAware接口都继承了Aware接口，我们来看下Aware接口的源码，如下所示：

```java
package org.springframework.beans.factory;

/**
 * A marker superinterface indicating that a bean is eligible to be notified by the
 * Spring container of a particular framework object through a callback-style method.
 * The actual method signature is determined by individual subinterfaces but should
 * typically consist of just one void-returning method that accepts a single argument.
 *
 * <p>Note that merely implementing {@link Aware} provides no default functionality.
 * Rather, processing must be done explicitly, for example in a
 * {@link org.springframework.beans.factory.config.BeanPostProcessor}.
 * Refer to {@link org.springframework.context.support.ApplicationContextAwareProcessor}
 * for an example of processing specific {@code *Aware} interface callbacks.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 */
public interface Aware {

}
```

可以看到，Aware接口是Spring 3.1版本中引入的接口，在Aware接口中，并未定义任何方法。

XxxAware的底层原理是由XxxAwareProcessor类实现的， 例如，我们这里以ApplicationContextAware接口为例，ApplicationContextAware接口的底层原理就是由ApplicationContextAwareProcessor类实现的。从ApplicationContextAwareProcessor类的源码可以看出，其实现了BeanPostProcessor接口，本质上都是后置处理器。

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor
```

初始化Spring容器时，也就是refresh()中有方法prepareBeanFactory()，该方法会向容器中注册ApplicationContextAwareProcessor后置处理器，Spring中的每个bean在初始化之前都会调用BeanPostProcessor.postProcessBeforeInitialization()方法，实现了XxxAware接口的类，就可以被注入Xxx对象的实例。

如下源码（注意：prepareBeanFactory(beanFactory);）：

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
            contextRefresh.end();
        }
    }
}
```

接着 prepareBeanFactory(beanFactory) 往下看：

```java

/**
* Configure the factory's standard context characteristics,
* such as the context's ClassLoader and post-processors.
* @param beanFactory the BeanFactory to configure
*/
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());
    if (!shouldIgnoreSpel) {
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    }
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {
        beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());
    }
}
```

**看一眼 ApplicationContextAwareProcessor** 这个类：

```java

/**
 * {@link BeanPostProcessor} implementation that supplies the {@code ApplicationContext},
 * {@link org.springframework.core.env.Environment Environment}, or
 * {@link StringValueResolver} for the {@code ApplicationContext} to beans that
 * implement the {@link EnvironmentAware}, {@link EmbeddedValueResolverAware},
 * {@link ResourceLoaderAware}, {@link ApplicationEventPublisherAware},
 * {@link MessageSourceAware}, and/or {@link ApplicationContextAware} interfaces.
 *
 * <p>Implemented interfaces are satisfied in the order in which they are
 * mentioned above.
 *
 * <p>Application contexts will automatically register this with their
 * underlying bean factory. Applications do not use this directly.
 *
 * @author Juergen Hoeller
 * @author Costin Leau
 * @author Chris Beams
 * @since 10.10.2003
 * @see org.springframework.context.EnvironmentAware
 * @see org.springframework.context.EmbeddedValueResolverAware
 * @see org.springframework.context.ResourceLoaderAware
 * @see org.springframework.context.ApplicationEventPublisherAware
 * @see org.springframework.context.MessageSourceAware
 * @see org.springframework.context.ApplicationContextAware
 * @see org.springframework.context.support.AbstractApplicationContext#refresh()
 */
class ApplicationContextAwareProcessor implements BeanPostProcessor {

	private final ConfigurableApplicationContext applicationContext;

	private final StringValueResolver embeddedValueResolver;


	/**
	 * Create a new ApplicationContextAwareProcessor for the given context.
	 */
	public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());
	}


	@Override
	@Nullable
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
				bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
				bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware ||
				bean instanceof ApplicationStartupAware)) {
			return bean;
		}

		AccessControlContext acc = null;

		if (System.getSecurityManager() != null) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
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
			((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
		}
		if (bean instanceof MessageSourceAware) {
			((MessageSourceAware) bean).setMessageSource(this.applicationContext);
		}
		if (bean instanceof ApplicationStartupAware) {
			((ApplicationStartupAware) bean).setApplicationStartup(this.applicationContext.getApplicationStartup());
		}
		if (bean instanceof ApplicationContextAware) {
			((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
		}
	}

}
```



在这之后，spring 初始化bean之前，也会判断这个bean是否实现了aware接口，并做相关处理（invokeAwareMethods）：

```java

	/**
	 * Initialize the given bean instance, applying factory callbacks
	 * as well as init methods and bean post processors.
	 * <p>Called from {@link #createBean} for traditionally defined beans,
	 * and from {@link #initializeBean} for existing bean instances.
	 * @param beanName the bean name in the factory (for debugging purposes)
	 * @param bean the new bean instance we may need to initialize
	 * @param mbd the bean definition that the bean was created with
	 * (can also be {@code null}, if given an existing bean instance)
	 * @return the initialized bean instance (potentially wrapped)
	 * @see BeanNameAware
	 * @see BeanClassLoaderAware
	 * @see BeanFactoryAware
	 * @see #applyBeanPostProcessorsBeforeInitialization
	 * @see #invokeInitMethods
	 * @see #applyBeanPostProcessorsAfterInitialization
	 */
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        // 注意看这里
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

下面是invokeAwareMethods方法：

```java
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```









