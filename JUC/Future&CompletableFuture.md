# Future



## 优缺点

### 优点

future + 线程池，异步多线程任务配合，能显著提高程序的执行效率。

### 缺点

futureTask.get() 方法容易导致阻塞。一般建议放到程序后面。

Future对于结果的获取不是很友好。只能通过阻塞（get）或者轮询（isDone）的方式得到任务的执行结果。

# CompletableFuture

## 为什么需要？

FutureTask 的 get()方法在Future 计算完成之前会一直处在阻塞状态下，isDone()方法容易耗费CPU资源，对于真正的异步处理我们希望是可以通过传入回调函数，在Future结束时自动调用该回调函数，这样，我们就不用等待结果。

阻塞的方式和异步编程的设计理念相违背，而轮询的方式会耗费无谓的CPU资源。因此，JDK8设计出CompletableFuture。

CompletableFuture提供了一种观察者模式类似的机制，可以让任务执行完成后通知监听的一方。

## CompletionStage是什么

代表异步计算过程中的某一个阶段，一个阶段完成以后可能会触发另外一个阶段，有些类似Linux系统的管道分隔符传参数。

## 如何通过CompletionStage创建一个异步任务？

不建议直接通过new的方式来创建CompletionStage对象。

一般通过CompletionStage提供的静态方法来创建对象。

```java
// 无返回值
public static CompletableFuture<Void> runAsync(Runnable runnable,
                                                   Executor executor) {
    return asyncRunStage(screenExecutor(executor), runnable);
}
// 有返回值
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                   Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}
```

## CompletableFuture的get方法/getNew方法/join方法有什么区别？

## CompletableFuture的thenApply与handle有什么区别？



## CompletableFuture的thenRun/thenApply/thenAccept有什么区别？



