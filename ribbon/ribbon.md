# Ribbon

**那么如果让你设计一个负载均衡组件，你会怎么设计？**

我们需要考虑这几个因素：

- 如何获取及同步服务器列表？涉及到与注册中心的交互。
- 如何将负载进行分摊？涉及到分摊策略。
- 如何将客户端请求进行拦截然后选择服务器进行转发？涉及到请求拦截。

抱着这几个问题，我们从负载均衡的原理 + Ribbon 的架构来学习如何设计一个负载均衡器，相信会带给你一些启发。

## 负载均衡

负载均衡的**两个基本点**：

- 选择哪个服务器来处理客户端请求。
- 将客户端请求转发出去。

**一个核心原理**：通过硬件或软件的方式维护一个服务列表清单。当用户发送请求时，会将请求发送给负载均衡器，然后根据负载均衡算法从可用的服务列表中选出一台服务器的地址，将请求进行转发，完成负载功能。

### 负载均衡的特性

**高性能**：可根据不同的分配规则自动将流量进行分摊。

**可扩展性**：可以很方便增加集群中设备或链路的数量。

**高可靠性**：系统中某个设备或链路发生故障，不会导致服务中断。

**易配置性**：配置和维护方便。

**透明性**：用户感知不到如何进行负载均衡的，也不用关心负载均衡。

### 负载均衡分类

负载均衡技术可以按照软件或硬件进行分类，也可以按照服务器列表存放的位置划分为服务端负载和客户端负载均衡。

#### 硬件负载均衡

F5 就是常见的硬件负载均衡产品。

优点：性能稳定，具备很多软件负载均衡不具备的功能，如应用交换，会话交换、状态监控等。

缺点：设备价格昂贵、配置冗余，没有软件负载均衡灵活，不能满足定制化需求。

#### 软件负载均衡

Nginx：性能好，可以负载超过 1W。工作在网络的7层之上，可以针对http应用做一些分流的策略。Nginx也可作为静态网页和图片服务器。Nginx仅能支持http、https和Email协议。

LVS（Linux Virtual Server）：是一个虚拟服务器集群系统，采用 IP 地址均衡技术和内容请求分发技术实现负载均衡。接近硬件设备的网络吞吐和连接负载能力。抗负载能力强、是工作在网络4层之上仅作分发之用。自身有完整的双机热备方案，如LVS+Keepalived。软件本身不支持正则表达式处理，不能做动静分离。

#### 服务端负载均衡

Nginx 和 F5 都可以划分到服务端的负载均衡里面，后端的服务器地址列表是存储在后端服务器中或者存在专门的 Nginx 服务器或 F5 上。

服务器的地址列表的来源是通过注册中心或者手动配置的方式来的。

#### 客户端负载均衡

终于轮到 Ribbon 登场了，它属于客户端负载均衡器，客户端自己维护一份服务器的地址列表。这个维护的工作就是由 Ribbon 来干的。

Ribbon 会从 Eureka Server 读取服务信息列表，存储在 Ribbon 中。如果服务器宕机了，Ribbon 会从列表剔除宕机的服务器信息。

Ribbon 有多种负载均衡算法，我们可以自行设定规则从而请求到指定的服务器。

## 均衡策略

### 轮循均衡（Round Robin）

原理：如果给服务器从 0 到 N 编号，轮询均衡策略会从 0 开始依次选择一个服务器作为处理本次请求的服务器。-

场景：适合所有父亲都有相同的软硬件配置，且请求频率相对平衡。

### 权重轮询均衡（Weighted Round Robin）

原理：按照服务器的不同处理能力，给服务器分配不同的权重，然后请求会按照权重分配给不同的服务器。

场景：服务器的性能不同，充分利用高性能的服务器，同时也能照顾到低性能的服务器。

### 随机均衡（Random）

原理：将请求随机分配给不同的服务器。

场景：适合客户端请求的频率比较随机的场景。

### 响应速度均衡（Response Time)

原理：负载均衡设备对每个服务器发送一个探测请求，看看哪台服务器的响应速度更快。

场景：适合服务器的响应性能不断变化的场景。

注意：响应速度是针对负载均衡设备和服务器之间的。

## Ribbon 核心组件

Ribbon 主要有六大功能组件：LoadBalancer、ServerList、Rule、Ping、ServerListFilter、ServerListUpdater。

![](img\ribbon01.png)	

### 负载均衡器 LoadBalancer

用于管理负载均衡的组件。初始化的时候通过加载 YMAL 配置文件创建出来的。

### 服务列表 ServerList

ServerList 主要用来获取所有服务的地址信息，并存到本地。

根据获取服务信息的方式不同，又分为静态存储和动态存储。

静态存储：从配置文件中获取服务节点列表并存储到本地。

动态存储：从注册中心获取服务节点列表并存储到本地

### 服务列表过滤 ServerListFilter

将获取到的服务列表按照过滤规则过滤。

- 通过 Eureka 的分区规则对服务实例进行过滤。
- 比较服务实例的通信失败数和并发连接数来剔除不够健康的实例。
- 根据所属区域过滤出同区域的服务实例。

### 服务列表更新 ServerListUpdater

服务列表更新就是 Ribbon 会从注册中心获取最新的注册表信息。是由这个接口 ServerListUpdater 定义的更新操作。而它有两个实现类，也就是有两种更新方式：

- 通过定时任务进行更新。由这个实现类 PollingServerListUpdater 做到的。
- 利用 Eureka 的事件监听器来更新。由这个实现类 EurekaNotificationServerListUpdater 做到的。

### 心跳检测 Ping

IPing 接口类用来检测哪些服务可用。如果不可用了，就剔除这些服务。

实现类主要有这几个：PingUrl、PingConstant、NoOpPing、DummyPing、NIWSDiscoveryPing。

心跳检测策略对象 IPingStrategy，默认实现是轮询检测。

### 负载均衡策略 Rule

Ribbon 的负载均衡策略和之前讲过的负载均衡策略有部分相同，先来个全面的图，看下 Ribbon 有哪几种负载均衡策略。

![](img\ribbon02.png)	

再来看下 Ribbon 源码中关于均衡策略的 UML 类图。

![](img\ribbon03.png)	

由图可以看到，主要由以下几种均衡策略：

- **线性轮询均衡** （RoundRobinRule）：轮流依次请求不同的服务器。优点是无需记录当前所有连接的状态，无状态调度。
- **可用服务过滤负载均衡**（AvailabilityFilteringRule）：过滤多次访问故障而处于断路器状态的服务，还有过滤并发连接数量超过阈值的服务，然后对剩余的服务列表按照轮询策略进行访问。默认情况下，如果最近三次连接均失败，则认为该服务实例断路。然后保持 30s 后进入回路关闭状态，如果此时仍然连接失败，那么等待进入关闭状态的时间会随着失败次数的增加呈指数级增长。
- **加权响应时间负载均衡**（WeightedResponseTimeRule）：为每个服务按响应时长自动分配权重，响应时间越长，权重越低，被选中的概率越低。
- **区域感知负载均衡**（ZoneAvoidanceRule）：更倾向于选择发出调用的服务所在的托管区域内的服务，降低延迟，节省成本。Spring Cloud Ribbon 中默认的策略。
- **重试负载均衡**（RetryRule)：通过轮询均衡策略选择一个服务器，如果请求失败或响应超时，可以选择重试当前服务节点，也可以选择其他节点。
- **高可用均衡**（BestAvailableRule)：忽略请求失败的服务器，尽量找并发比较低的服务器。注意：这种会给服务器集群带来成倍的压力。
- **随机负载均衡**（RandomRule）：随机选择服务器。适合并发比较大的场景。



## Ribbon 拦截请求的原理

负载均衡器如何将客户端请求进行拦截然后选择服务器进行转发？

结合上面介绍的 Ribbon 核心组件，我们可以画一张原理图来梳理下 Ribbon 拦截请求的原理：

![](img\ribbon04.png)	

第一步：Ribbon 拦截所有标注`@loadBalance`注解的 RestTemplate。RestTemplate 是用来发送 HTTP 请求的。

第二步：将 Ribbon 默认的拦截器 LoadBalancerInterceptor 添加到 RestTemplate 的执行逻辑中，当 RestTemplate 每次发送 HTTP 请求时，都会被 Ribbon 拦截。

第三步：拦截后，Ribbon 会创建一个 ILoadBalancer 实例。

第四步：ILoadBalancer 实例会使用 RibbonClientConfiguration 完成自动配置。就会配置好 IRule，IPing，ServerList。

第五步：Ribbon 会从服务列表中选择一个服务，将请求转发给这个服务。

## Ribbon 初始化的原理

当我们去剖析 Ribbon 源码的时候，需要找到一个突破口，而 @LoadBalanced 注解就是一个比较好的入口。

先来一张 Ribbon 初始化的流程图：

![](img\ribbon05.png)	

添加注解的代码如下所示：

```java
@LoadBalanced
@Bean
public RestTemplate getRestTemplate() {
 return new RestTemplate();
}
```

第一步：Ribbon 有一个自动配置类 LoadBalancerAutoConfiguration，SpringBoot 加载自动配置类，就会去初始化 Ribbon。

第二步：当我们给 RestTemplate 或者 AsyncRestTemplate 添加注解后，Ribbon 初始化时会收集加了 @LoadBalanced 注解的 RestTemplate 和 AsyncRestTemplate ，把它们放到一个 List 里面。

第三步：然后 Ribbon 里面的 RestTemplateCustomizer 会给每个 RestTemplate 进行定制化，也就是加上了拦截器：LoadBalancerInterceptor。

第四步：从 Eureka 注册中心获取服务列表，然后存到 Ribbon 中。

第五步：加载 YMAL 配置文件，配置好负载均衡配置，创建一个 ILoadbalancer 实例。



## Ribbon 同步服务列表原理

Ribbon 首次从 Eureka 获取全量注册表后，就会隔一定时间获取注册表。原理图如下：

![](img\ribbon06.png)		

之前我们提到过 Ribbon 的核心组件 ServerListUpdater，用来同步注册表的，它有一个实现类 PollingServerListUpdater ，专门用来做定时同步的。默认1s 后执行一个 Runnable 线程，后面就是每隔 30s 执行 Runnable 线程。这个 Runnable 线程就是去获取 Eureka 注册表的。

## Eureka 心跳检测的原理

我们知道 Eureka 注册中心是通过心跳检测机制来判断服务是否可用的，如果不可用，可能会把这个服务摘除。为什么是可能呢？因为 Eureka 有自我保护机制，如果达到自我保护机制的阀值，后续就不会自动摘除。

这里我们可以再复习下 Eureka 的自我保护机制和服务摘除机制。

- **Eureka 心跳机制**：每个服务每隔 30s 自动向 Eureka Server 发送一次心跳，Eureka Server 更新这个服务的最后心跳时间。如果 180 s 内（版本1.7.2）未收到心跳，则任务服务故障了。
- **Eureka 自我保护机制**：如果上一分钟实际的心跳次数，比我们期望的心跳次数要小，就会触发自我保护机制，不会摘除任何实例。期望的心跳次数：服务实例数量 * 2 * 0.85。
- **Eureka 服务摘除机制**：不是一次性将服务实例摘除，每次最多随机摘除 15%。如果摘除不完，1 分钟之后再摘除。

说完 Eureka 的心跳机制和服务摘除机制后，我们来看下 Ribbon 的心跳机制。

## Ribbon 心跳检测的原理

Ribbon 的心跳检测原理和 Eureka 还不一样，**Ribbon 不是通过每个服务向 Ribbon 发送心跳或者 Ribbon 给每个服务发送心跳来检测服务是否存活的**。

先来一张图看下 Ribbon 的心跳检测机制：

![](img\ribbon07.png)	

Ribbon 心跳检测原理：对自己本地缓存的 Server List 进行遍历，看下每个服务的状态是不是 UP 的。具体的代码就是 isAlive 方法。

核心代码：

```java
isAlive = status.equals(InstanceStatus.UP);
```

那么多久检测一次呢？

默认每隔 30s 执行以下 PingTask 调度任务，对每个服务执行 isAlive 方法，判断下状态。

## Ribbon 常用配置项

### 禁用 Eureka

```properties
# 禁用 Eureka
ribbon.eureka.enabled=false
```

服务注册列表默认是从 Eureka 获取到的，如果不想使用 Eureka，可以禁用掉。然后我们需要手动配置服务列表。

### 配置服务列表

```properties
ribbon-config-passjava.ribbon.listOfServers=localhost:8081,localhost:8083
```

这个配置是针对具体服务的，前缀就是服务名称，配置完之后就可以和之前一样使用服务名称来调用接口了。

### 其他配置项

![](img\ribbon08.png)	

## 总结

本篇深入讲解了 Spring Cloud 微服务中 负载均衡组件 Ribbon 架构原理，分为几大块：

- Ribbon 的六大核心组件
- Ribbon 如何拦截请求并进行转发的。
- Ribbon 初始化的原理。
- Ribbon 如何同步 Eureka 注册表的原理。
- Eureka 和 Ribbon 两种 心跳检测的原理
- Ribbon 的常用配置项。

https://www.cnblogs.com/zhixiang-org-cn/archive/2019/10/31/11769320.html

https://my.oschina.net/u/3748584/blog/4814474


























