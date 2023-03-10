# IO 模型的基础知识

`I/O` 请求可以分为两个阶段，分别为调用阶段和执行阶段。

- 第一个阶段为 **I/O 调用阶段**，即用户进程向内核发起系统调用。
- 第二个阶段为 **I/O 执行阶段**。此时，内核等待 `I/O` 请求处理完成返回。该阶段分为两个过程：首先等待数据就绪，并写入内核缓冲区；随后将内核缓冲区数据拷贝至用户态缓冲区。

如何理解准备数据阶段呢？对于写请求：等待系统调用的完整请求数据，并写入内核缓冲区；对于读请求：等待系统调用的完整请求数据；（若请求数据不存在于内核缓冲区）则将外围设备的数据读入到内核缓冲区。

无论是Java还是其他的语言，本质上`IO`读写操作的原理是类似的，编程语言开发的程序，一般都是工作在用户态空间，但由于`IO`读写对于计算机而言，属于高危操作，所以`OS`不可能`100%`将这些功能开放给用户态的程序使用，所以正常情况下的程序读写操作，本质上都是在调用`OS`内核提供的函数：`read()、 write()`。
 也就是说，在程序中试图利用`IO`机制读写数据时，仅仅只是调用了内核提供的接口函数而已，本质上真正的`IO`操作还是由内核自己去完成的。

`IO`工作的过程如下：

![](img\IO.jpg)	

在上述中提到了一个`CPU`中断信号的概念，这其实属于一种`I/O`的控制方式，`IO`控制方式目前主要有三种：**忙等待方式、中断驱动方式以及`DMA`直接存储器方式**，不过无论是何种方式，本质上的最终作用是相同的，都是读取数据。

在上述`IO`工作过程中，其实大体可分为两部分：**准备阶段和复制阶段**，准备阶段是指数据从网络网卡或本地存储器读取到内核的过程，而复制阶段则是将内核缓冲区中的数据拷贝至用户态的进程缓冲区。常听的`BIO、NIO、AIO`之间的区别，就在于这两个过程中的操作是同步还是异步的，是阻塞还是非阻塞的。




# Linux的5种主要IO模式

## 同步阻塞IO（BIO）

![](img\BIO.jpg)

如上图所表现的那样，应用进程向内核发起 `I/O` 请求，发起调用的线程一直等待内核返回结果。一次完整的 `I/O` 请求称为 `BIO`（`Blocking IO`，阻塞 `I/O`），所以 `BIO` 在实现异步操作时，**只能使用多线程模型**，一个请求对应一个线程。但是，线程的资源是有限且宝贵的，创建过多的线程会增加线程切换的开销。

其实可以使用线程池解决线程过多的情况，但这就意味着，线程池的最大数量就是服务端可以支撑的最大并发数量。

可以联想到 `Java` 的 `Socket`：

- 服务端会有两个阻塞操作：一个是 `accept()`，也就是等待客户端连接，如果没有客户端连接，就会阻塞；第二个是 `getInputStream()`，如果客户端连接上了，但是没有发送数据，也是会阻塞的。
- 客户端获取数据也就是 `getInputStream()`，也是会阻塞的。

## 同步非阻塞 I/O（NIO）

![](img\NIO.jpg)

如上图所示，应用进程向内核发起 `I/O` 请求后不再会同步等待结果，而是会立即返回，通过轮询的方式获取请求结果。

NIO 与原来的 I/O 有同样的作用和目的, 他们之间最重要的区别是数据打包和传输的方式。原来的 I/O 以流的方式处理数据，而 NIO 以块的方式处理数据。

`NIO` 相比 `BIO` 虽然大幅提升了性能，但是轮询过程中大量的系统调用导致上下文切换开销很大（内核态与用户态的切换）。所以，**单独使用非阻塞 I/O 时效率并不高**，而且随着并发量的提升，非阻塞 `I/O` 会存在严重的性能浪费。

**为什么轮询会导致上下文开销很大呢？**

因为轮询操作是需要用户态切换到核心态的，这样是很耗费性能的。

## I/O多路复用

![](img\IO多路复用.jpg)

多路复用实现了**一个线程处理多个 I/O 句柄的操作**。多路指的是多个**数据通道**，复用指的是使用一个或多个固定线程来处理每一个 `Socket`。`select、poll、epoll` 都是 `I/O 多路复用`的具体实现，线程一次 `select` 调用可以获取内核态中多个数据通道的数据状态。

多路复用解决了同步阻塞 I/O 和同步非阻塞 I/O 的问题，是一种非常高效的 I/O 模型。

其实说简单点，就是**把轮询的操作放到内核中进行**了，这样就可以避免大量的用户态和内核态进行切换，`select、poll、epoll` 都是 `Linux` 内核提供的函数，目的就是在内核中遍历所有的文件描述符（在 `Linux` 中，一切皆文件，所以 `Socket` 被抽象成了文件，简称 fd）。

`Redis` 采用的就是 `IO` 多路复用模型。

### select/poll

Select是内核提供的系统调用，它支持一次查询多个系统调用的可用状态，当任意一个结果状态可用时就会返回，用户进程再发起一次系统调用进行数据读取。换句话说，就是NIO中N次的系统调用，借助Select，只需要发起一次系统调用就够了。

但是，select有一个限制，就是存在连接数限制，针对于此，又提出了poll。其与select相比，主要是解决了连接限制。

select/poll 虽然解决了NIO重复无效系统调用用的问题，但同时又引入了新的问题。问题是：

1. 用户空间和内核空间之间，大量的数据拷贝
2. 内核循环遍历IO状态，浪费CPU时间

换句话说，select/poll虽然减少了用户进程的发起的系统调用，但内核的工作量只增不减。在高并发的情况下，内核的性能问题依旧。所以select/poll的问题本质是：内核存在无效的循环遍历。

### epoll

针对select/pool引入的问题，我们把解决问题的思路转回到内核上，如何减少内核重复无效的循环遍历呢？**变主动为被动，基于事件驱动来实现。**

epoll相较于select/poll，多了两次系统调用，其中epollcreate建立与内核的连接，epollctl注册事件，epoll_wait阻塞用户进程，等待IO事件。

## 信号驱动 I/O

![](img\信号驱动IO.jpg)

信号驱动 I/O 并不常用，它是一种**半异步的 I/O 模型**。在使用信号驱动 I/O 时，当数据准备就绪后，内核通过发送一个 SIGIO 信号通知应用进程，应用进程就可以开始读取数据了。

信号驱动IO与BIO和NIO最大的区别就在于，在IO执行的数据准备阶段，不会阻塞用户进程。如下图所示：当用户进程需要等待数据的时候，会向内核发送一个信号，告诉内核我要什么数据，然后用户进程就继续做别的事情去了，而当内核中的数据准备好之后，内核立马发给用户进程一个信号，说”数据准备好了，快来查收“，用户进程收到信号之后，立马调用recvfrom，去查收数据。

乍一看，信号驱动式I/O模型有种异步操作的感觉，但是在IO执行的第二阶段，也就是将数据从内核空间复制到用户空间这个阶段，用户进程还是被阻塞的。

综上，你会发现，不管是BIO还是NIO还是SIGIO，它们最终都会被阻塞在IO执行的第二阶段。那如果能将IO执行的第二阶段变成非阻塞，那就完美了。

## 异步I/O（AIO）

![](img/AIO.jpg)

异步IO真正实现了IO全流程的非阻塞。用户进程发出系统调用后立即返回，内核等待数据准备完成，然后将数据拷贝到用户进程缓冲区，然后发送信号告诉用户进程**IO操作执行完毕**（与SIGIO相比，一个是发送信号告诉用户进程数据准备完毕，一个是IO执行完毕）。

异步 I/O 最重要的一点是从内核缓冲区拷贝数据到用户态缓冲区的过程也是由系统异步完成，应用进程只需要在指定的数组中引用数据即可。

**异步 I/O 与信号驱动 I/O 这种半异步模式的主要区别**：信号驱动 I/O 由内核通知何时可以开始一个 I/O 操作，而异步 I/O 由内核通知 I/O 操作何时已经完成。

# JDK IO

Java 共支持3种网络编程的I/O模型：BIO、NIO、AIO。

在 JDK 1.4 投入使用之前，只有 BIO 一种模式。开发过程相对简单。新来一个连接就会创建一个新的线程处理。随着请求并发度的提升，BIO 很快遇到了性能瓶颈。

JDK 1.4 以后开始引入了 NIO 技术，支持 select 和 poll；

JDK 1.5 支持了 epoll；

JDK 1.7 发布了 NIO2，支持 AIO 模型。Java 在网络领域取得了长足的进步。

## BIO、NIO、AIO适用场景分析

BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序简单易理解。

NIO方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，弹幕系统，服务器间通讯等。编程比较复杂，JDK1.4开始支持。

AlO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务制充分调用OS参与并发操作，编程比较复杂，JDK7开始支持。







































