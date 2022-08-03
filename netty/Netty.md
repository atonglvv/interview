# Netty是什么

在 `Netty` 官网首页有这样一句话：

```
Netty is an asynchronous event-driven network application framework for rapid development of maintainable high performance protocol servers & clients.
```

翻译：Netty 是一个异步事件驱动的网络应用程序框架用于快速开发可维护的高性能协议服务器和客户端。

# 为什么选择Netty？

`Netty` 大大简化了网络应用的开发过程。

选择netty的三个原因：

- 高性能，低延迟；

- 完美弥补 NIO 的缺陷；

- 更低的资源消耗。

## 高性能，低延迟

Netty 的 I/O 模型是基于非阻塞 I/O （NIO）实现的，底层依赖的是 JDK NIO 框架的多路复用器 Selector。一个多路复用器 Selector 可以同时轮询多个 Channel，采用 epoll 模式后，只需要一个线程负责 Selector 的轮询，就可以接入成千上万的客户端。

在 I/O 多路复用的场景下，当有数据处于就绪状态后，需要一个事件分发器（Event Dispather），它负责将读写事件分发给对应的读写事件处理器（Event Handler）。事件分发器有两种设计模式：Reactor 和 Proactor，**Reactor 采用同步 I/O， Proactor 采用异步 I/O**。

Reactor 实现相对简单，适合处理耗时短的场景，对于耗时长的 I/O 操作容易造成阻塞。Proactor 性能更高，但是实现逻辑非常复杂，目前主流的事件驱动模型还是依赖 selector 或 epoll 来实现。

Netty 所采用的主从 Reactor 多线程模型，所有的 I/O 事件都注册到一个 I/O 多路复用器上，当有 I/O 事件准备就绪后，I/O 多路复用器会将该 I/O 事件通过事件分发器分发到对应的事件处理器中。

该线程模型避免了同步问题以及多线程切换带来的资源开销，真正做到高性能、低延迟。

## 完美弥补 Java NIO 的缺陷

Netty 是基于 JDK NIO 的，相比于 NIO，Netty 做的更多，并且做得更好。

**易用性。** 我们使用 JDK NIO 编程需要了解很多复杂的概念，比如 Channels、Selectors、Sockets、Buffers 等，编码复杂程度令人发指。相反，Netty 在 NIO 基础上进行了更高层次的封装，屏蔽了 NIO 的复杂性；Netty 封装了更加人性化的 API，统一的 API（阻塞/非阻塞） 大大降低了开发者的上手难度；与此同时，Netty 提供了很多开箱即用的工具，例如常用的行解码器、长度域解码器等，而这些在 JDK NIO 中都需要你自己实现。

**稳定性。** Netty 更加可靠稳定，修复和完善了 JDK NIO 较多已知问题，例如臭名昭著的 select 空转导致 CPU 消耗 100%，TCP 断线重连，keep-alive 检测等问题。

**可扩展性。** Netty 的可扩展性在很多地方都有体现，这里我主要列举其中的两点：一个是可定制化的线程模型，用户可以通过启动的配置参数选择 Reactor 线程模型；另一个是可扩展的事件驱动模型，将框架层和业务层的关注点分离。大部分情况下，开发者只需要关注 ChannelHandler 的业务逻辑实现。

## 更低的资源消耗

作为网络通信框架，需要处理海量的网络数据，那么必然面临有大量的网络对象需要创建和销毁的问题，对于 JVM GC 并不友好。为了降低 JVM 垃圾回收的压力，Netty 主要采用了两种优化手段：

- **对象池复用技术。** Netty 通过复用对象，避免频繁创建和销毁带来的开销。
- **零拷贝技术。** 除了操作系统级别的零拷贝技术外，Netty 提供了更多面向用户态的零拷贝技术，例如 Netty 在 I/O 读写时直接使用 DirectBuffer，从而避免了数据在堆内存和堆外内存之间的拷贝。

# 网络框架的选型

**Tomcat 作为一款非常优秀的 Web 服务器看上去已经帮我们解决了类似问题，那么它与 Netty 到底有什么不同？**

- Netty 和 Tomcat 最大的区别在于对**通信协议**的支持，可以说 Tomcat 是一个 HTTP Server，它主要解决 HTTP 协议层的传输，而 Netty 不仅支持 HTTP 协议，还支持 SSH、TLS/SSL 等多种应用层的协议，而且能够自定义应用层协议。
- Tomcat 需要遵循 Servlet 规范，在 Servlet 3.0 之前采用的是同步阻塞模型，Tomcat 6.x 版本之后已经支持 NIO，性能得到较大提升。然而 Netty 与 Tomcat 侧重点不同，所以不需要受到 Servlet 规范的约束，可以最大化发挥 NIO 特性。
- 如果你仅仅需要一个 HTTP 服务器，那么我推荐你使用 Tomcat。术业有专攻，Tomcat 在这方面的成熟度和稳定性更好。但如果你需要做面向 TCP 的网络应用开发，那么 Netty 才是你最佳的选择。

**为什么 Tomcat 不选择 Netty 来做通信层呢？**

- 历史原因，Tomcat 是 1998 年出现的，Netty 是 2004 年出现的，也就是说 Tomcat 出现的时候还没有 Netty。
- 其实很多情况都是这样，历史原因导致有些技术选型或者框架已经无法改变了，并不说其他的技术不够好。

## 比较出名的网络框架还有 Mina 和 Grizzly。

- Mina 是 Apache Directory 服务器底层的 NIO 框架，由于 Mina 和 Netty 都是 Trustin Lee 主导的作品，所以两者在设计理念上基本一致。Netty 出现的时间更晚，可以认为是 Mina 的升级版，解决了 Mina 一些设计上的问题。比如 Netty 提供了可扩展的编解码接口、优化了 ByteBuffer 的分配方式，让用户使用起来更为便捷、安全。
- Grizzly 出身 Sun 公司，从设计理念上看没有 Netty 优雅，几乎是对 Java NIO 比较初级的封装，目前业界使用的范围也很小。

综上所述，Netty 是我们一个较好的选择。

# 谁在用Netty？

`Netty` 经过很多出名产品在线上的大规模验证，其健壮性和稳定性都被业界认可，其中典型的产品有以下几个：

- 服务治理：`Apache Dubbo、gRPC`。
- 大数据：`Hbase、Spark、Flink、Storm`。
- 搜索引擎：`Elasticsearch`。
- 消息队列：`RocketMQ、ActiveMQ`。





























