# dubbo核心功能

基于接口代理

服务自动注册与发现

软负载均衡

容错与监控

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

