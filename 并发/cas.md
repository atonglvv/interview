# CAS

CAS 在 java.util.concurrent.atomic 相关类、Java AQS、CurrentHashMap 等实现上有非常广泛的应用。比如，在 AtomicInteger 的实现中，静态字段 valueOffset 即为字段 value 的内存偏移地址，valueOffset 的值在 AtomicInteger 初始化时，在静态代码块中通过 Unsafe 的 objectFieldOffset 方法获取。在 AtomicInteger 中提供的线程安全方法中，通过字段 valueOffset 的值可以定位到 AtomicInteger 对象中 value 的内存地址，从而可以根据 CAS 实现对 value 字段的原子操作。



# CAS操作系统层面实现

compareAndSet

Atomic::cmpxchg

LOCK_IF_MP

lock cmpxchgq (原子操作)



# 如何解决ABA问题

添加一个version（版本号），操作的时候需要给version+1。