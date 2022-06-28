# Overview

ThreadLocal提供了一种与众不同的线程安全方式，它不是通过锁（悲观、乐观）来保证线程安全，而是彻底的避免了冲突的发生。

ThreadLocal的作用主要是做数据隔离，填充的数据只属于当前线程，变量的数据对别的线程而言是相对隔离的。

# ThreadLocal Demo

# Thread 与 ThreadLocal

**ThreadLocal 是线程 Thread 中属性 threadLocals 的管理者。**

对于ThreadLocal的 get，set，remove的操作结果都是针对当前线程Thread实例的threadLocals存，取，删除操作。

在 `Thread` 类中有维护两个 `ThreadLocal.ThreadLocalMap` 对象：`threadLocals` 和 `inheritableThreadLocals` （初始为 null，只有在调用 `ThreadLocal` 类的 set 或 get 时才创建它们）。

`ThreadLocalMap` 是  `ThreadLocal` 的内部类，其 key 为弱引用的 `ThreadLocal` 对象，value 为对应设置的 Object 对象。

让我们先看一下 Thread 类：

```java
public class Thread implements Runnable {
    
	/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    
    /**
     * Initializes a Thread.
     * 
     * @param g the Thread group
     * @param target the object whose run() method gets called
     * @param name the name of the new Thread
     * @param stackSize the desired stack size for the new thread, or
     *        zero to indicate that this parameter is to be ignored.
     * @param acc the AccessControlContext to inherit, or
     *            AccessController.getContext() if null
     * @param inheritThreadLocals if {@code true}, inherit initial values for
     *            inheritable thread-locals from the constructing thread
     */
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
    
    /**
     * This method is called by the system to give a Thread
     * a chance to clean up before it actually exits.
     */
    private void exit() {
        if (group != null) {
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
    }
}
```



# ThreadLocal源码分析

三个成员变量

```java
public class ThreadLocal<T> {
    /**
     * ThreadLocals rely on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     */
    private final int threadLocalHashCode = nextHashCode();

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;
}
```





## ThreadLocalMap

`ThreadLocalMap` 的数据结构实际上是数组，对比 `HashMap` 它只有散列数组没有链表。（这个地方就会涉及到Hash冲突的解决方案问题）

















# 八股文

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

![关系图](\关系图.jpg)

## ThreadLocalMap 是如何解决hash冲突的？

线性探测

## Entry的key为什么存ThreadLocal的弱引用？

## Entry的value不是弱引用对象，如果其key回收了，value没回收会造成什么问题？如何避免？

尽量避免大对象的value。



# ThreadLocal 应用场景

## 多数据源的切换

## spring声明式事务



