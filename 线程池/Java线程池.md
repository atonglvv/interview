# 简单说一下你对线程池的理解

池化技术的思想主要是为了减少每次获取资源的消耗，提高对资源的利用率。

有以下好处：

- **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- **提高响应速度**。当任务到达时，任务可以不需要等到线程创建就能立即执行。
- **提高线程的可管理性**。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

# 讲一下Executor 

`Executor` 框架不仅包括了线程池的管理，还提供了线程工厂、队列以及拒绝策略。

其子类`ThreadPoolExecutor` 在实际线程池的使用过程中，使用频率非常高。

# ThreadPoolExecutor主要参数

**`ThreadPoolExecutor` 3 个最重要的参数：**

- **`corePoolSize` :** 核心线程数线程数定义了最小可以同时运行的线程数量。核心线程会一直存在，即使没有任务执行。
- **`maximumPoolSize` :** 当队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数。
- **`workQueue`:** 当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

`ThreadPoolExecutor`其他常见参数 :

1. **`keepAliveTime`**:当线程池中的线程数量大于 `corePoolSize` 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 `keepAliveTime`才会被回收销毁；
2. **`unit`** : `keepAliveTime` 参数的时间单位。
3. **`threadFactory`** :executor 创建新线程的时候会用到。
4. **`handler`** :拒绝策略。

# ThreadPoolExecutor的拒绝策略(饱和策略)

- **`ThreadPoolExecutor.AbortPolicy`** ：抛出 `RejectedExecutionException`来拒绝新任务的处理。
- **`ThreadPoolExecutor.CallerRunsPolicy`** ：谁调用谁执行，由发起者线程执行，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。
- **`ThreadPoolExecutor.DiscardPolicy`** ：不处理新任务，直接丢弃掉。不抛异常。
- **`ThreadPoolExecutor.DiscardOldestPolicy`** ： 丢弃最早的（等待时间最长的）未处理的任务请求。



# 为什么建议使用`ThreadPoolExecutor` 声明线程池？而不是**`Executors`** ？

Executors 返回线程池对象的弊端如下：

- **FixedThreadPool**和 **SingleThreadExecutor** ： 允许请求的队列长度为 Integer.MAX_VALUE,可能堆积大量的请求，从而导致 OOM。
- **CachedThreadPool 和 ScheduledThreadPool** ： 允许创建的线程数量为 `Integer.MAX_VALUE` ，可能会创建大量线程，从而导致 OOM。



# 如何确认合适的线程数量？

如果是CPU密集型应用，则线程池大小设置为N+1 （N为CPU总核数）

如果是IO密集型应用，则线程池大小设置为2N+1 （N为CPU总核数）

# 线程池场景

