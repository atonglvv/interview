# Spring声明周期

Class

推断构造方法

实例化

依赖注入

初始化前（@PostConstruct）

初始化（判断Bean有没有实现InitializingBean接口，并重写afterPropertiesSet()方法）

初始化后（Aop）

代理对象（代理对象跟普通对象只能有一个放入到单例池，代理对象没有进行依赖注入）

放入单例池Map

使用

销毁

# spring与tomcat的关系

tomcat是一类Web应用服务器，Spring 框架是一个轻量级的 Java 开发框架。Tomcat提供Web容器服务，Spring为应用开发提供平台。

tomcat在启动时会触发容器初始化事件 。

tomcat启动时会加载和解析web.xml文件，获取其中配置的filter,servlet，listener等设置到context中后，接下来就是执行了。执行到配置的ContextLoaderListener时，该监听器最终会调用AbstractApplicationContext.refresh（）便会启动Spring容器。

spring的ContextLoaderListener监听到这个事件后会执⾏contextinitialized⽅法，在该⽅法中会初始化spring 的根容器即ioc容器，初始化完成之后，会将这个容器放⼊到servletContext中，以便获取 。

tomcat在启动过程中还会去扫描加载servlet,⽐如springmvc的dispatchServlet(前端控制器),⽤来处理每⼀个 servlet 。 

 servlet⼀般采⽤延时加载,当⼀个请求过来的时候发现dispatchservlet还未初始化,则会调⽤其init⽅法,初始化 时候会建⽴⾃⼰的容器spring mvc容器,同时spring mvc容器通过servletcontext上下⽂来获取到spring ioc容 器将ioc容器设置为其⽗容器; 

注意：spring mvc中容器可以访问spring ioc容器中的bean反之不能即在controller中可以注⼊service bean对象,在service中不能注⼊controller容器。

# aop代理对象没有进行依赖注入，那对象的依赖从哪获取？

cglib生成代理对象，通过字节码生成被代理对象的子类，并且依赖被代理对象，这样就可以在代理对象中使用被代理对象的依赖。

# Spring Bean 如果有两个有参构造方法，可以成功创建Bean么？

不可以。除非在某个构造方法上添加**@Autowired**。

如果只有一个构造方法是可以成功创建bean。

如果有多个有参构造方法跟一个无参构造方法，则默认使用无参构造，可以成功创建bean。

如果多个有参构造，没有无参构造，则会报错。可以在构造方法上添加@Autowired，告诉spring用哪个构造方法创建bean。

# Autowired 怎么从IOC中获取Bean？

先byType，后byName。

# PopulateBean？

# 考虑一个问题，A Bean被代理，A Bean依赖于B Bean，直接从IOC get A Bean，获取的A Bean 的属性B有值么？

没有， 此时A中的B为null。

# 填充Bean的属性环节的循环依赖问题

# 什么是循环依赖？如何解决？

将实例化过程 跟 初始化过程拆分开，是解决循环依赖问题的关键。

三级缓存，一级缓存中放的完整对象，二级缓存中放的非完成对象（未初始化）。

## 为什么需要三级缓存？

什么是三级缓存？

第一级缓存：单例池 singletonObjects

第二级缓存：earlySingletonObjects

第三级缓存：

## 只用一级缓存有什么问题？

只用 单例池 singletonObjects ，无法解决循环依赖问题。如果A依赖B且B依赖A，创建A bean 的时候需要去创建B bean，创建Bbean的时候又需要A的bean。

## 假如只用两级缓存有什么问题？（即，将实例化的对象放到map里）

加了二级缓存，将实例化后的普通对象放到二级缓存里。

这种操作可以解决循环依赖问题。

但是在bean生命周期，初始化后会有aop，生成代理对象操作。

真正依赖的bean，应该是代理对象，而不是普通对象。

只用一二级缓存无法实现代理对象的切面功能。

## 缓存的放置时间和删除时间

# aware接口是干嘛用的？

# BeanPostProcessor的后置处理方法，关于AOP

# BeanFactory跟FactoryBean有什么区别？

相同点：都能用来创建bean对象。



# Spring的设计模式

单例模式：bean默认都是单例的

工厂模式：BeanFactory

模板模式：PostProcessBeanFactory, onRefresh

策略模式：XmlBeanDefinitionReader, PropertiesBeanDefinitionReader

观察者模式：listener，event，multicast

适配器模式：

装饰者模式：BeanWrapper

责任链模式：aop 拦截器链

代理模式：动态代理

# SpringAop底层实现原理

bean的创建过程中有一个步骤可以对bean进行扩展实现[BeanPostProcessor]，aop本身就是一个扩展功能，所以在BeanPostProcessor的后置处理方法中来进行实现。

# Spring事务代理对象的切面逻辑

Spring的事务是由aop实现的。

Transactionlnterceptor.invoke()

1. 解析方法上的事务属性(@Transactionl)，根据属性判断是否开启新事务
2. 当需要开启的时候，获取数据库连接，关闭自动提交功能（conn.autocommit = false），开启事务
3. 执行业务sql
4. 操作过程中，如果失败了，通过completeTransactionAfterThrowing完成事务的回滚，回滚的具体逻辑是doRollBack
5. 操作过程中，如果成功，通过commitTransactionAfterReturning完成事务的提交
6. 当事务执行完毕，需要清除相关的事务信息cleanupTransactionlnfo

# Spring事务传播

## 传播特性有几种？

在日常工作中用required，required_new,nested是用的比较多的。



# 事务失效问题？

## 方法调用

```java
@Service
public class UserService {

    @Transactional
    public void a () {
        //此处省略 业务逻辑sql
        b();
    }

    @Transactional
    public void b() {
        //此处省略 业务逻辑sql
    }
}
```

如上代码，a方法调用本类b方法，事务失效。

原因：事务方法是由代理对象来执行，但是a()方法调用b()方法则是由普通对象调用执行，普通对象不会关心b()方法上的@Transactional，所以不会考虑事务。

## 多线程

spring事务底层是由ThreadLocal<Map<DataSource,  conn>>实现的，无法保证多个线程间的事务性。





# Spring的体系结构

Core

Context

web

mvc

dao

orm

aop



# @ComponentScan() 的includeFilters与excludeFilters

# Spring的作用域

## Spring有哪些作用域？

prototype

singleton

request ： 主要针对web应用，一次请求创建一个实例。

session：一个会话，创建一个实例。

## 单实例与多实例

模式是单实例

创建容器的时候，单实例Bean就创建完成，并放到spring容器里。

多实例Bean只有在getBean的时候才会创建。

使用@Scope来声明Bean的作用域。

# @Lazy 懒加载

主要作用于单实例bean，懒加载，使容器启动时不创建对象，仅当第一次使用Bean的时候才创建实例并初始化。

# @Conditional

自定义Condition Class 需要实现Condition接口，重写matches方法。

使用：@Conditional(consumeConditional.class)



# @Import与ImportSelector接口与ImportBeanDefinitionRegistrar接口





# 给容器中注册组件的几种方式？

xml

@Bean

@ComponentScan

@Import

FactoryBean



# 如果容器中存在多个相同类型的Bean，获取bean的优先级是什么？



# @Primary 注解的作用？

如果容器中存在多个相同类型的bean，则优先获取@Primary注解的bean。





# spring与tomcat的关系

toma在启动时会触发容器初始化事件 。

spring的contextLoaderListener监听到这个事件后会执⾏contextinitialized⽅法，在该⽅法中会初始化spring 的根容器即ioc容器，初始化完成之后，会将这个容器放⼊到servletContext中，以便获取 。

tomcat在启动过程中还会去扫描加载servlet,⽐如springmvc的dispatchServlet(前端控制器),⽤来处理每⼀个 servlet 。 

 servlet⼀般采⽤延时加载,当⼀个请求过来的时候发现dispatchservlet还未初始化,则会调⽤其init⽅法,初始化 时候会建⽴⾃⼰的容器spring mvc容器,同时spring mvc容器通过servletcontext上下⽂来获取到spring ioc容 器将ioc容器设置为其⽗容器; 

注意：spring mvc中容器可以访问spring ioc容器中的bean反之不能即在controller中可以注⼊service bean对象,在service中不能注⼊controller容器。



# 了解过cglib动态代理吗？他跟jdk动态代理的区别是什么？

优先是jdk动态代理，其次是cglib动态代理。



# spring的扩展机制

## BeanFactoryPostProcessor

BeanFactoryPostProcessor，是由 Spring 框架组建提供的容器扩展机制，允许在 Bean 对象注册后但未实例化之前，对 Bean 的定义信息 `BeanDefinition` 执行修改操作。Bean必须实现 BeanFactoryPostProcessor 接口，并重写postProcessBeanFactory方法，如下代码：

```java
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        BeanDefinition beanDefinition = beanFactory.getBeanDefinition("userService");
        PropertyValues propertyValues = beanDefinition.getPropertyValues();

        propertyValues.addPropertyValue(new PropertyValue("age", 28));
    }

}
```

## BeanPostProcessor

BeanPostProcessor，也是 Spring 提供的扩展机制，不过 BeanPostProcessor 是在 Bean 对象实例化之后修改 Bean 对象，也可以替换 Bean 对象。这部分与后面要实现的 AOP 有着密切的关系。Bean必须实现 BeanPostProcessor 接口，并重写postProcessBeforeInitialization 跟 postProcessAfterInitialization 方法，如下代码：

```java
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if ("userService".equals(beanName)) {
            UserService userService = (UserService) bean;
            userService.setLocation("改为：北京");
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

}
```











