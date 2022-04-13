# ThreadLocal

ThreadLocal提供了一种与众不同的线程安全方式，它不是在发生线程冲突时想办法解决冲突，而是彻底的避免了冲突的发生。

ThreadLocal的作用主要是做数据隔离，填充的数据只属于当前线程，变量的数据对别的线程而言是相对隔离的。

ThreadLocal和lock都可以解决线程安全问题，但是ThreadLocal并不能解决共享变量的并发问题。而且他们的使用场景不同，关键看你的资源是需要多线程之间共享的还是单线程内部共享的。

## ThreadLocal Demo



## Thread 与 ThreadLocal 与 ThreadLocalMap 与 Entry之间的关系

Thread类下有两个ThreadLocalMap成员变量，如下：

```java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

ThreadLocalMap 是 ThreadLocal 的静态内部类。

Entry 是 ThreadLocalMap 的静态内部类。Entry 的key 是ThreadLocal 类型的，value 是Object 类型。

关系图如下：

![image-20210918145254131](C:\Users\carver\AppData\Roaming\Typora\typora-user-images\image-20210918145254131.png)



## ThreadLocal 应用



