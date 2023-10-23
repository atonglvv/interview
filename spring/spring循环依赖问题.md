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

























