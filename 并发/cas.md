# CAS操作系统层面实现

compareAndSet

Atomic::cmpxchg

LOCK_IF_MP

lock cmpxchgq (原子操作)



# 如何解决ABA问题

添加一个version（版本号），操作的时候需要给version+1。