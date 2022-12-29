# dubbo核心功能

基于接口代理

服务自动注册与发现

软负载均衡

容错与监控

# 调用方式

## 同步调用（Sync）

服务消费者发起RPC调用后，线程会一直阻塞，直到服务提供者返回结果或者超时异常。在RPC框架中，一般会采用同步调用的方式，但是在RPC框架内部的实现中，本质上还是采用的是异步处理。例如，**Dubbo中就存在着经典的异步转同步的逻辑**。另外，当采用同步调用方式时，要设置超时时间。如下图所示。

![](\img\sync.png)	

## 异步调用（Future）

服务消费者发起RPC调用后，线程不会阻塞，获取到RPC框架返回的Future对象。RPC调用的结果数据会被服务提供者缓存起来，服务消费者根据自身实际情况决定获取结果数据。当服务消费者主动获取异步结果数据时是阻塞的。如下图所示。

![](\img\future.png)	

## 回调（Callback）

服务消费者发起RPC调用后，会将回调接口Callback对象发送给RPC框架。此时也不需要同步等待RPC框架的返回结果，直接返回。当RPC框架接收到服务提供者处理的结果数据或者超时异常后，会执行Calback回调。一般在定义和实现Callback接口时，都要实现处理成功的方法和异常方法。例如 success(Obiect result) 方法和 fai(Exception e)方法。

![](\img\callback.png)	

## 单向调用（Oneway）

服务消费者发送RPC调用后，直接返回，忽略返回结果。

![](\img\oneway.png)

# dubbo通信协议

dubbo（默认，port：20880）

hessian

rmi

http

webservice

thrift

memcached

redis

# dobbo 动态代理怎么实现的？

JDK动态代理

Javassist代理



# dubbo consumer timeout 跟 provider timeout

consumer timeout 优先

![time](\img\time.jpg)

# dubbo 的重试次数

幂等接口（查询、修改、删除）可以设置重试次数，非幂等接口（新增）不要设置重试次数（retries=0）



# zookeeper宕机，consumer还能不能访问provider？

能，consumer有本地缓存。

# 没有注册中心，consumer能连provider么？

可以，可以通过直连方式。

@Reference(url="127.0.0.1:20880")



# dubbo的负载均衡策略

RandomLoadBalance：基于权重的随机负载均衡。是Dubbo的**默认**负载均衡策略。

RoundRobinLoadBalance：轮询负载均衡。轮询选择一个。

LeastActiveLoadBalance：最少活跃调用数，相同活跃数的随机。活跃数指调用前后计数差。使处理请求慢的 Provider 收到更少请求，因为越慢的 Provider 的调用前后计数差会越大。

ConsistentHashLoadBalance：一致性哈希负载均衡。相同参数的请求总是落在同一台机器上。

# dubbo的容错与降级

通过dubbo-admin可视化界面实现



# dubbo的服务暴露

ServiceBean implements InitializingBean，ApplicationListener<ContextRefreshedEvent>

当然，ServiceBean不只实现了这两个接口，只能说这两个接口比较重要！

ServiceBean在容器创建完对象的时候，会执行InitializingBean下的afterPropertiesSet方法。将xml配置的接口信息保存起来，包括配置的timeout，protocol，application，module，registry，monitor。。。

ServiceBean在IOC容器启动完成后，调用ApplicationListener下的onApplicationEvent方法。在这个方法里，调用export方法-->doExport

