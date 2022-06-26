# 锁升级的过程

## 为什么需要偏向锁？

考虑如果大部分时间只有一个线程去请求资源，这时候还搞重量级锁那一套（monitor对象，开启阻塞队列），有点大材小用（浪费资源）。

所以引入了偏向锁。

# 偏向锁，代码执行完后会释放对象头MarkWord中的Thread_Id么？

不会的。

下次如果该线程再获取锁的时候，可以直接执行。



# 轻量级锁-重量级锁的过程？以及自适应自旋的理解



# synchronized和lock的区别

- lock是一个接口，synchronized是一个关键字，synchronized是内置的语言实现
- synchronized在发生异常时候会自动释放锁，不会出现死锁；lock发生异常时候，不会主动释放锁，必须手动unlock来释放锁。
- lock更加灵活，比如可以中断等待，trylock知道有没有获取锁，还可以实现读写分离（readwritelock）



# 一个类里如果有多个synchronized方法，同一个对象可以同时访问两个不同的synchronized方法么？为什么？

不可以