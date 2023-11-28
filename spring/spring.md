# spring与tomcat的关系

toma在启动时会触发容器初始化事件 。

spring的contextLoaderListener监听到这个事件后会执⾏contextinitialized⽅法，在该⽅法中会初始化spring 的根容器即ioc容器，初始化完成之后，会将这个容器放⼊到servletContext中，以便获取 。

tomcat在启动过程中还会去扫描加载servlet,⽐如springmvc的dispatchServlet(前端控制器),⽤来处理每⼀个 servlet 。 

 servlet⼀般采⽤延时加载,当⼀个请求过来的时候发现dispatchservlet还未初始化,则会调⽤其init⽅法,初始化 时候会建⽴⾃⼰的容器spring mvc容器,同时spring mvc容器通过servletcontext上下⽂来获取到spring ioc容 器将ioc容器设置为其⽗容器; 

注意：spring mvc中容器可以访问spring ioc容器中的bean反之不能即在controller中可以注⼊service bean对象,在service中不能注⼊controller容器。



# 了解过cglib动态代理吗？他跟jdk动态代理的区别是什么？

优先是jdk动态代理，其次是cglib动态代理。



# spring循环依赖

## Spring如何解决循环依赖问题的？

三级缓存。

spring内部有三级缓存：

- singletonObjects 一级缓存，用于保存初始化完成的bean实例
- earlySingletonObjects 二级缓存，用于保存半成品，只实例化的Bean
- singletonFactories 三级缓存，用于保存`ObjectFactory`对象，以便于后面扩展有机会创建代理对象。



![001](\img\001.jpg)



**循环依赖解决的流程：**

实例化A对象

将lambda表达式（匿名内部类）放到三级缓存中

填充A属性（需要B Bean）

先从一级缓存中找B Bean，一级缓存找不到再找二级缓存，二级找不到再找三级缓存

没有发现 B Bean， 则需要CreateBean B（创建B Bean）

实例化B对象

将lambda表达式（匿名内部类，生产B Bean的函数）放到三级缓存中

填充B属性（需要A Bean）

先从一级缓存中找A Bean，一级缓存找不到再找二级缓存，二级找不到再找三级缓存

从三级缓存中找到A Bean 的lambda表达式，执行得到A Bean，并将A Bean（半成品）放到二级缓存中

填充B属性完成

初始化 B Bean

将B Bean 放到一级缓存中（将三级缓存中的Factory删除），并将B Bean 返回给 A，填充A的B属性

初始化A

将ABean 从二级缓存删除，放到一级缓存中



## 用两级缓存为什么不行？

如果对象被代理，会有两个Bean，一个是普通Bean，一个是代理Bean。IOC容器是将 代理Bean放到一级缓存中。

如果 A 依赖于 B， B 依赖于A，且 A 需要代理。如果只用两级缓存，那B Bean 注入的 A Bean 就不是代理Bean，而是普通A Bean。

所以需要三级缓存 提前暴露 代理。

























