# 什么是aop

AOP，即**Aspect Oriented Rrogramming**的缩写，意为：面向切面编程，通过预编译方式和运行期间动态代理实现程序功能统一维护的一种技术。AOP可以对业务漏极的各个部分进行隔离，从而使得业务逻辑之间得耦合性降低，提高程序的可重用性，同时提高了开发得效率。

AOP和OOP是两种不同的设计思想。OOP（面向对象编程）针对业务处理过程得实体及其属性和行为进行抽象封装，获得清晰高效得逻辑单元划分。AOP则是针对业务处理过程中得切面进行提取，是面对业务处理过程中的某个步骤或阶段，获得逻辑过程中各部分之间低耦合性得隔离效果。

面向切面编程的好处就是：减少重复，专注业务。它是面向对象编程的一种补充。

# AOP的基本概念（Spring的专业术语）

增强：向各个程序内部注入一些逻辑代码从而增强原有程序的功能。

连接点（JoinPoint）：类中可以被增强的方法，这个方法就就被称为连接点，切记连接点并不是一定会被增强。

切入点（Pointcut）：类中实际被增强的方法。

通知（Advice）：指一个切面在特定的连接点要做的事情，简单来说就是“增强”。可以分为方法执行前通知，方法执行后通知，环绕通知等等。

切面（Aspect）：把通知添加到切入点的过程就叫切面。

目标（Target）：代理的目标对象，即要增强的方法所在的类。

代理（Proxy）：向目标对象应用通知之后创建的代理对象。

# aop原理

首先来看看Spring是如何集成AspectJ AOP的，这时候目光应该定格在在注解`@EnableAspectJAutoProxy`上

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

 /**
  * Indicate whether subclass-based (CGLIB) proxies are to be created as opposed
  * to standard Java interface-based proxies. The default is {@code false}.
  * 该属性指定是否通过CGLIB进行动态代理，spring默认是根据是否实现了接口来判断使用JDK还是CGLIB,
  * 这也是两种代理主要区别，如果为ture，spring则不做判断直接使用CGLIB代理
  */
 boolean proxyTargetClass() default false;

 /**
  * Indicate that the proxy should be exposed by the AOP framework as a {@code ThreadLocal}
  * for retrieval via the {@link org.springframework.aop.framework.AopContext} class.
  * Off by default, i.e. no guarantees that {@code AopContext} access will work.
  * @since 4.3.1
  * 暴露aop代理，这样就可以借助ThreadLocal特性在AopContext在上下文中获取到，可用于解决内部方法调用 a()
  * 调用this.b()时this不是增强代理对象问题，通过AopContext获取即可
  */
 boolean exposeProxy() default false;

}
```

该注解核心代码：`@Import(AspectJAutoProxyRegistrar.class)`，这也是`Spring`集成其他功能通用方式了。来到`AspectJAutoProxyRegistrar`类：

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

 /**
  * Register, escalate, and configure the AspectJ auto proxy creator based on the value
  * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
  * {@code @Configuration} class.
  */
    @Override
    public void registerBeanDefinitions(
        AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        // 重点 重点 重点   注入AspectJAnnotationAutoProxyCreator
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

        // 根据@EnableAspectJAutoProxy注解属性进行代理方式和是否暴露aop代理等设置
        AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }
            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }
    }

}
```

可以看到这里`@import`就是作用就是将`AnnotationAwareAspectJAutoProxyCreator`注册到容器当中，**这个类是Spring AOP的关键**。

![](D:\ainterview\interview\spring\img\aop01.png)从上面类图可以看出它本质上是一种`BeanPostProcessor`。所以它的执行是在完成原始 Bean 构建后的初始化Bean（initializeBean）过程中进行代理对象生成的，最终放到`Spring`容器中，我们可以看下它的postProcessAfterInitialization 方法，该方法在其上级父类中`AbstractAutoProxyCreator`实现的：

```java
/**
  * Create a proxy with the configured interceptors if the bean is
  * identified as one to proxy by the subclass.
  * @see #getAdvicesAndAdvisorsForBean
  */
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

可以看到只有当`earlyProxyReferences`集合中不存在`cacheKey`的时候，才会执行`wrapIfNecessary`方法。Spring AOP对象生成的时机有两个：一个是提前AOP，提前AOP的对象会被放入到`earlyProxyReferences`集合当中，Spring循环依赖解决方案中如果某个bean有循环依赖，同时需要代理增强，那么就会提前生成aop代理对象放入`earlyProxyReferences`中，若没有提前，AOP会在Bean的生命周期的最后执行`postProcessAfterInitialization`的时候进行AOP动态代理。

进入`#wrapIfNecessary()`方法，核心逻辑：

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {

    .......

        // Create proxy if we have advice.
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

`#getAdvicesAndAdvisorsForBean()`会判断bean是否有advisor，判断bean上是否加了`@Aspect`注解，对加了该注解的类再判断其拥有的所有方法，对于加了**通知注解**的方法构建出`Advisor`通知对象放入候选通知链当中。接着基于当前加载的Bean通过切点表达式筛选通知，添加ExposeInvocationInterceptor拦截器，最后对通知链进行排序，得到最终的通知链。得到完整的advice通知链信息后，紧接着通过`#createProxy()`生成代理对象。

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
                             @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    // new 一个代理工厂对象
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    // 判断使用JDK代理还是CGLIB代理
    if (proxyFactory.isProxyTargetClass()) {
        // Explicit handling of JDK proxy targets (for introduction advice scenarios)
        if (Proxy.isProxyClass(beanClass)) {
            // Must allow for introductions; can't just set interfaces to the proxy's interfaces only.
            for (Class<?> ifc : beanClass.getInterfaces()) {
                proxyFactory.addInterface(ifc);
            }
        }
    }
    else {
        // No proxyTargetClass flag enforced, let's apply our default checks...
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // 构建advisor通知链
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    // 将通知链放入代理工厂
    proxyFactory.addAdvisors(advisors);
    // 将被代理的目标类放入到代理工程中
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
    // 基于代理工厂获取代理对象返回
    return proxyFactory.getProxy(getProxyClassLoader());
}
```

\#proxyFactory.getProxy(getProxyClassLoader())

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```

\#createAopProxy()

```java
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                                         "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```

AOP的代理方式有两种，一种是CGLIB代理，使用`ObjenesisCglibAopProxy`来创建代理对象，另一种是JDK动态代理，使用`JdkDynamicAopProxy`来创建代理对象，最终通过对应的`AopProxy`的`#getProxy()`生成代理对象，来看看`JdkDynamicAopProxy`的：

```java
@Override
public Object getProxy() {
    return getProxy(ClassUtils.getDefaultClassLoader());
}

@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
    if (logger.isTraceEnabled()) {
        logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
    }
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

`Proxy.newProxyInstance(classLoader, proxiedInterfaces, this)`这不就是JDK动态代理技术实现模板代码嘛。到这里aop代理对象就已经生成放到`Spring`容器中。

接下来我们就来看看AOP是怎么执行增强方法的，也就是如何执行aspect切面的通知方法的？还是以JDK实现的动态代理`JdkDynamicAopProxy`为例，其实现了`InvocationHandler`来实现动态代理，意味着调用代理对象的方法会调用`JdkDynamicAopProxy`中的`invoke()`方法，核心逻辑如下：

```java
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    try {

        .......

            Object retVal;

        if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            // 暴露aop代理对象
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        // Get as late as possible to minimize the time we "own" the target,
        // in case it comes from a pool.
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);

        // Get the interception chain for this method.
        // 根据切面通知，获取方法的拦截链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        // Check whether we have any advice. If we don't, we can fallback on direct
        // reflective invocation of the target, and avoid creating a MethodInvocation.
        // 拦截链为空，直接反射调用目标方法
        if (chain.isEmpty()) {
            // We can skip creating a MethodInvocation: just invoke the target directly
            // Note that the final invoker must be an InvokerInterceptor so we know it does
            // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        }
        else {
            // 如果不为空，那么会构建一个MethodInvocation对象，调用该对象的proceed()方法执行拦截器链以及目标方法
            // We need to create a method invocation...
            MethodInvocation invocation =
                new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            // Proceed to the joinpoint through the interceptor chain.
            retVal = invocation.proceed();
        }

        // Massage return value if necessary.
        Class<?> returnType = method.getReturnType();
        if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
            // Special case: it returned "this" and the return type of the method
            // is type-compatible. Note that we can't help if the target sets
            // a reference to itself in another returned object.
            retVal = proxy;
        }
        else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
            throw new AopInvocationException(
                "Null return value from advice does not match primitive return type for: " + method);
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            // Must have come from TargetSource.
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```



# aop相关问题

## @AfterReturning、@After以及AfterThrowing的区别

```java
/*
 * @Before和@Around的执行顺序 :
 * 若是基于XML 则 其结果 受影响于 配置的先后顺序。
 */

/*
 * @AfterReturning和@After的区别 :
 * (1) @AfterReturning 被代理的方法执行完成之后 要执行的代码。
 * (2) @After 新生成的代理方法执行完成之后 要执行的代码，是放在finally块中的。
 * 注: 若是基于XML 则 两者的执行顺序结果 受影响于 配置的先后顺序。
 */

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object result;
    try {
        //@Before
        result = method.invoke(target, args);
        //@AfterReturning
        return result;
    } catch (InvocationTargetException e) {
        Throwable targetException = e.getTargetException();
        //@AfterThrowing
        throw targetException;
    } finally {
        //@After
    }
}
```

