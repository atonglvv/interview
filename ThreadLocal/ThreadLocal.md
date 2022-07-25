# Overview

ThreadLocal提供了一种与众不同的线程安全方式，它不是通过锁（悲观、乐观）来保证线程安全，而是彻底的避免了冲突的发生。

ThreadLocal的作用主要是做数据隔离，填充的数据只属于当前线程，变量的数据对别的线程而言是相对隔离的。

# 适用场景（为什么要用ThreadLocal）

- 场景1，ThreadLocal 用作保存每个**线程独享**的对象，为每个线程都创建一个副本，这样每个线程都可以修改自己所拥有的副本, 而不会影响其他线程的副本，确保了线程安全。

- 场景2，ThreadLocal 用作每个线程内需要独立保存信息，以便供其他方法更方便地获取该信息的场景。每个线程获取到的信息可能都是不一样的，前面执行的方法保存了信息后，后续方法可以通过 ThreadLocal 直接获取到，避免了传参，类似于**全局变量**的概念。

# ThreadLocal Demo

一个小Demo，From ThreadLocal 源码注释。来说明线程间的数据隔离。

```java
import java.util.concurrent.atomic.AtomicInteger;

/**
 * @program: Draft
 * @description: ThreadLocal 源码注释Demo
 * @author: atong
 * @create: 2022-06-27 16:22
 */
public class ThreadId {
    // Atomic integer containing the next thread ID to be assigned
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId =
            new ThreadLocal<Integer>() {
                @Override protected Integer initialValue() {
                    return nextId.getAndIncrement();
                }
            };

    private static final ThreadLocal<Integer> THREAD_LOCAL = new ThreadLocal<>();

    // Returns the current thread's unique ID, assigning it if necessary
    public static int get() {
        return threadId.get();
    }

    public static void add() {
        threadId.set(threadId.get() + 1);
    }

    public static void remove() {
        threadId.remove();
    }

    public static void incrementSameThreadId(int loop) {
        try {
            for (int i = 0; i < loop; i++) {
                ThreadId.add();
                System.out.println(Thread.currentThread() + "_" + i + ", threadId: " + ThreadId.get());
            }
        } finally {
            ThreadId.remove();
        }
    }

    public static void main(String[] args) {
        incrementSameThreadId(3);
        new Thread(() -> ThreadId.incrementSameThreadId(2)).start();
        new Thread(() -> ThreadId.incrementSameThreadId(5)).start();
    }
}
```

## 可优化的地方

在给ThreadLocal赋初始值的那块代码可优化如下：

```java
    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId =
            new ThreadLocal<Integer>() {
                @Override protected Integer initialValue() {
                    return nextId.getAndIncrement();
                }
            };
```

可优化为 **withInitial** ：

```java
    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId =
            ThreadLocal.withInitial(nextId::getAndIncrement);
```

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

## 三个成员变量

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

## initialValue

注意该方法的修饰符 **protected** （允许子类继承）

```java
    /**
     * Returns the current thread's "initial value" for this
     * thread-local variable.  This method will be invoked the first
     * time a thread accesses the variable with the {@link #get}
     * method, unless the thread previously invoked the {@link #set}
     * method, in which case the {@code initialValue} method will not
     * be invoked for the thread.  Normally, this method is invoked at
     * most once per thread, but it may be invoked again in case of
     * subsequent invocations of {@link #remove} followed by {@link #get}.
     *
     * <p>This implementation simply returns {@code null}; if the
     * programmer desires thread-local variables to have an initial
     * value other than {@code null}, {@code ThreadLocal} must be
     * subclassed, and this method overridden.  Typically, an
     * anonymous inner class will be used.
     *
     * @return the initial value for this thread-local
     */
    protected T initialValue() {
        return null;
    }
```

## setInitialValue

```java
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

## set

```java
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        // 初始化一个ThreadLocalMap。后面会分析 ThreadLocalMap 源码。
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

## get

```java
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // ThreadLocalMap的key就是当前ThreadLocal对象实例
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 如果map没有初始化，那么在这里初始化一下
        return setInitialValue();
    }

    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

## withInitial And SuppliedThreadLocal

```java
    /**
     * Creates a thread local variable. The initial value of the variable is
     * determined by invoking the {@code get} method on the {@code Supplier}.
     *
     * @param <S> the type of the thread local's value
     * @param supplier the supplier to be used to determine the initial value
     * @return a new thread local variable
     * @throws NullPointerException if the specified supplier is null
     * @since 1.8
     */
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }

    /**
     * An extension of ThreadLocal that obtains its initial value from
     * the specified {@code Supplier}.
     */
    static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

        private final Supplier<? extends T> supplier;

        SuppliedThreadLocal(Supplier<? extends T> supplier) {
            this.supplier = Objects.requireNonNull(supplier);
        }

        @Override
        protected T initialValue() {
            return supplier.get();
        }
    }
```

## ThreadLocalMap

`ThreadLocalMap` 的数据结构实际上是数组，对比 `HashMap` 它只有散列数组没有链表。（这个地方就会涉及到Hash冲突的解决方案问题）

### 四个属性以及一个静态内部类

注意 **Entry** extends **WeakReference** 。

然后在Entry的构造函数里调用了一下super(key)。

所以 Entry的key就是一个弱引用对象。

```java
    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0
    }

```

### 构造函数

```java

        /**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            // 计算数组下标
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            // 设置 size
            size = 1;
            // 设置阈值 扩容用
            setThreshold(INITIAL_CAPACITY);
        }

        /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        /**
         * Construct a new map including all Inheritable ThreadLocals
         * from given parent map. Called only by createInheritedMap.
         *
         * @param parentMap the map associated with parent thread.
         */
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
```

### nextIndex

```java
		/**
         * Increment i modulo len.
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
```

### set

```java

        /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            // 根据key计算index
            int i = key.threadLocalHashCode & (len-1);
			
            // 如果index下Entry不为空
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    // 覆盖之前的value值
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }


        /**
         * Replace a stale entry encountered during a set operation
         * with an entry for the specified key.  The value passed in
         * the value parameter is stored in the entry, whether or not
         * an entry already exists for the specified key.
         *
         * As a side effect, this method expunges all stale entries in the
         * "run" containing the stale entry.  (A run is a sequence of entries
         * between two null slots.)
         *
         * @param  key the key
         * @param  value the value to be associated with key
         * @param  staleSlot index of the first stale entry encountered while
         *         searching for key.
         */
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

```

### getEntry

```java
        /**
         * Get the entry associated with key.  This method
         * itself handles only the fast path: a direct hit of existing
         * key. It otherwise relays to getEntryAfterMiss.  This is
         * designed to maximize performance for direct hits, in part
         * by making this method readily inlinable.
         *
         * @param  key the thread local object
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

        /**
         * Version of getEntry method for use when key is not found in
         * its direct hash slot.
         *
         * @param  key the thread local object
         * @param  i the table index for key's hash code
         * @param  e the entry at table[i]
         * @return the entry associated with key, or null if no such
         */
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    // key 已经被回收.entry无效
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

        /**
         * Expunge a stale entry by rehashing any possibly colliding entries
         * lying between staleSlot and the next null slot.  This also expunges
         * any other stale entries encountered before the trailing null.  See
         * Knuth, Section 6.4
         *
         * @param staleSlot index of slot known to have null key
         * @return the index of the next null slot after staleSlot
         * (all between staleSlot and this slot will have been checked
         * for expunging).
         * 删除没用的 entry
         */
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }

```

# InheritableThreadLocal

在实际开发过程中，我们可能会遇到这么一种场景。

主线程开了一个子线程，但是我们希望在子线程中可以访问主线程中的ThreadLocal对象，也就是说有些数据需要进行父子线程间的传递。比如像这样：

```java
public static void main(String[] args) {
    ThreadLocal threadLocal = new ThreadLocal();
    IntStream.range(0,10).forEach(i -> {
        // 每个线程的序列号，希望在子线程中能够拿到
        threadLocal.set(i);
        // 这里来了一个子线程，我们希望可以访问上面的threadLocal
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + ":" + threadLocal.get());
        }).start();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
}
```

如果我们希望子线可以看到父线程的ThreadLocal，那么就可以使用InheritableThreadLocal。顾名思义，这就是一个支持线程间父子继承的ThreadLocal，将上述

代码中的 ThreadLocal 使用 InheritableThreadLocal ：

```java
InheritableThreadLocal threadLocal = new InheritableThreadLocal();
```

虽然InheritableThreadLocal看起来挺方便的，但是依然要注意以下几点：

- 变量的传递是发生在线程创建的时候，如果不是新建线程，而是用了线程池里的线程，就不灵了。
- 变量的赋值就是从主线程的map复制到子线程，它们的value是同一个对象，如果这个对象本身不是线程安全的，那么就会有线程安全问题。

# 八股文

## ThreadLocalMap的数据结构

就是一个数组。

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

## 为什么用ThreadLocal做key而不是使用Thread做key？

一个线程中只使用了一个`ThreadLocal`对象，那么使用`Thread`做key也未尝不可。

但实际情况中，一个线程中很有可能不只使用了一个ThreadLocal对象。这时使用`Thread`做key就会有问题了。

## Entry的key为什么存ThreadLocal的弱引用？

这样设计的好处是，如果这个变量不再被其他对象使用时，可以自动回收这个ThreadLocal对象，避免可能的内存泄露（注意，Entry中的value，依然是强引用）。

## 了解ThreadLocal的内存泄漏问题么？（value内存泄漏）

虽然ThreadLocalMap中的key是弱引用，当不存在外部强引用的时候，就会自动被回收，但是Entry中的value依然是强引用。如下图：

![对象引用](\对象引用.jpg)

只有当Thread被回收时，这个value才有被回收的机会，否则，只要线程不退出，value总是会存在一个强引用。但是，对于线程池来说，大部分线程会一直存在在系统的整个生命周期内，那样的话，就会造成value对象出现泄漏的可能。处理的方法是，在ThreadLocalMap调用 set(), get(), remove() 的时候，都要清理无用的value。

在set()， get()， remove() 方法中，都会直接或者间接调用到**expungeStaleEntry**方法进行value的清理。

但是，ThreadLocal并不能100%保证不发生内存泄漏。

比如，很不幸的，你的get()方法总是访问固定几个一直存在的ThreadLocal，那么清理动作就不会执行，如果你没有机会调用set()和remove()，那么这个内存泄漏依然会发生。

因此，一个良好的习惯依然是：**当你不需要这个ThreadLocal变量时，主动调用remove()，这样对整个系统是有好处的**。

**同时也要尽量避免大对象的value。**

# ThreadLocal 应用场景

## 多数据源的切换

## spring声明式事务

## MDC 日志框架

## SimpleDateFormat 线程安全解决方案

### SimpleDateFormat 线程不安全分析

>  [还在用 SimpleDateFormat 做时间格式化？小心项目崩掉！](https://mp.weixin.qq.com/s/JBDpKpn3efuYQ4ilce7tZw)

`SimpleDateFormat` 继承 `DateFormat` 类，`SimpleDateFormat`转换日期是通过继承自`DateFormat`类的`Calendar`对象来操作的，`Calendar`对象会被用来进行日期-时间计算，既被用于`format`方法也被用于`parse`方法。

#### DateFormat

```java
public abstract class DateFormat extends Format {

    /**
     * The {@link Calendar} instance used for calculating the date-time fields
     * and the instant of time. This field is used for both formatting and
     * parsing.
     *
     * <p>Subclasses should initialize this field to a {@link Calendar}
     * appropriate for the {@link Locale} associated with this
     * <code>DateFormat</code>.
     * @serial
     */
    protected Calendar calendar;
}
```

#### parse

代码太多了。这里不粘贴复制了。可以自行查看 parse(source, pos)。

`Calendar`对象它并不是线程安全的，如果，a线程将`calendar`清空了，`calendar` 就没有新值了，恰好此时b线程刚好进入到parse方法用到了`calendar`对象，那就会产生线程安全问题了！

```java

    /**
     * Parses text from the beginning of the given string to produce a date.
     * The method may not use the entire text of the given string.
     * <p>
     * See the {@link #parse(String, ParsePosition)} method for more information
     * on date parsing.
     *
     * @param source A <code>String</code> whose beginning should be parsed.
     * @return A <code>Date</code> parsed from the string.
     * @exception ParseException if the beginning of the specified string
     *            cannot be parsed.
     */
    public Date parse(String source) throws ParseException
    {
        ParsePosition pos = new ParsePosition(0);
        Date result = parse(source, pos);
        if (pos.index == 0)
            throw new ParseException("Unparseable date: \"" + source + "\"" ,
                pos.errorIndex);
        return result;
    }


```



#### format

在执行 `SimpleDateFormat.format()` 方法时，会使用 `calendar.setTime()` 方法将输入的时间进行转换，那么我们想想一下这样的场景：

- 线程 1 执行了 `calendar.setTime(date)` 方法，将用户输入的时间转换成了后面格式化时所需要的时间；
- 线程 1 暂停执行，线程 2 得到 CPU 时间片开始执行；
- 线程 2 执行了 `calendar.setTime(date)` 方法，对时间进行了修改；
- 线程 2 暂停执行，线程 1 得出 CPU 时间片继续执行，因为线程 1 和线程 2 使用的是同一对象，而时间已经被线程 2 修改了，所以此时当线程 1 继续执行的时候就会出现线程安全的问题了。

### ThreadLocal 解决

```java
public class SimpleDateFormatThreadLocalDemo {

    private static ThreadLocal<SimpleDateFormat> simpleDateFormat = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static void main(String[] args) {
        while (true) {
            try {
                new Thread(() -> {
                    String dateStr = simpleDateFormat.get().format(new Date());
                    try {
                        Date parseDate = simpleDateFormat.get().parse(dateStr);
                        String dateStrCheck = simpleDateFormat.get().format(parseDate);
                        boolean equals = dateStr.equals(dateStrCheck);
                        if (!equals) {
                            System.out.println(equals + " " + dateStr + " " + dateStrCheck);
                        } else {
                            System.out.println(equals);
                        }
                    } catch (ParseException e) {
                        System.out.println(e.getMessage());
                    }
                }).start();
            } finally {
                simpleDateFormat.remove();
            }
        }
    }
}
```







