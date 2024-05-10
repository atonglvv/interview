# Spring事务传播机制

Spring 事务传播机制是指，包含多个事务的方法在相互调用时，事务是如何在这些方法间传播的。

既然是“事务传播”，所以事务的数量应该在两个或两个以上，Spring 事务传播机制的诞生是为了规定多个事务在传播过程中的行为的。比如方法 A 开启了事务，而在执行过程中又调用了开启事务的 B 方法，那么 B 方法的事务是应该加入到 A 事务当中呢？还是两个事务相互执行互不影响，又或者是将 B 事务嵌套到 A 事务中执行呢？所以这个时候就需要一个机制来规定和约束这两个事务的行为，这就是 Spring 事务传播机制所解决的问题。

## REQUIRED（default）

REQUIRED(Spring默认的事务传播类型 required)：如果当前没有事务，则自己新建一个事务，如果当前存在事务则加入这个事务。

当A调用B的时候：如果A中没有事务，B中有事务，那么B会新建一个事务；如果A中也有事务、B中也有事务，那么B会加入到A中去，变成一个事务，这时，要么都成功，要么都失败。

```java
class ServiceA {
    @Autowired 
    ServiceB serviceB；
    
    @Transactional
    void a() {
        ....
        serviceB.b();
        ....
    }
}

class ServiceB {
    @Transactional
    void b() {
        ......
    }
}
```

1. 线程执行到serviceA.a() 方法时，其实是执行的  代理serviceA对象的a方法。
2. 执行代理serviceA对象的a方法
    2.1: 执行a方法的增强逻辑-> 事务增强器 (环绕增强)
    2.2: 事务增强器会做什么事? 提取事务标签属性
    2.3: 检查当前线程有没有绑定 conn 数据库连接 资源？ 发现当前线程未绑定（TransactionSync...Manager#resources 是 ThreadLocal<Map<obj,obj>>，检查key:datasource 有没有数据）
    2.4: 因为未绑定conn资源，所以线程下一步就是 到 datasource.getConnection() 获取一个conn资源
    2.5: 因为新获取的conn资源的autocommit是true，所以这一步 修改 autocommit 为false，表示手动提交事务，这一步也表示 开启事务（修改conn其它 属性..）
    2.6: 绑定conn资源到 TransactionSync...Manager#resources，key：datasource
3. 执行事务增强器后面的增强器..
4. 最后一个advice调用 target的目标方法 a() 方法
    4.1: 假设target a方法 需要访问数据库 执行SQL 的话，程序需要获取一个 conn 资源，到哪拿？ DataSourceUtils.getConnection(datasource) 这一步最终会拿到 事务增强器 前置增强逻辑 存放在 TransactionSync..Manager#resources 内的conn 资源
    4.2: 执行方法a逻辑...可能会执行一些 SQL 语句...
5. 线程执行到这样一行代码：serviceB.b()
6. serviceB 它是一个代理对象，因为它也使用了 @Transactional 注解了，Spring 会为它创建代理的。
7. 执行代理serviceB对象的b方法
    7.1: 执行b方法的增强逻辑-> 事务增强器（环绕增强）
    7.2: 事务增强器会做什么事? 提取事务标签属性
    7.3: 检查当前线程有没有绑定 conn 数据库连接 资源？发现当前线程已经绑定了 conn 数据库连接资源了
    7.4: 检查事务注解属性，发现自己打的propagation == REQUIRED，所以继续共享 conn 数据库链接资源
8. 执行事务增强器后面的增强器..
9. 最后一个device调用 target (serviceB)的目标方法 b() 方法
    9.1: 假设target b方法 需要访问数据库 执行SQL 的话，程序需要获取一个 conn 资源，到哪拿？ DataSourceUtils.getConnection(datasource) 这一步最终会拿到 代理serviceA对象存放在 TransactionSync..Manager#resources 内的conn 资源
    9.2: 执行方法b逻辑...可能会执行一些 SQL 语句...
10. 线程继续执行 事务增强器 环绕增强的后置逻辑 （代理serviceB.b() 方法的 后置增强）
     10.1: 检查发现，serviceB.b() 事务并不是 当前 b方法开启的，所以 基本不做什么事情..
11. 线程继续回到 目标 serviceA.a() 方法内，继续执行
     11.1: 执行方法a逻辑...可能会执行一些 SQL 语句...
12. 线程继续回到 代理 serviceA.a() 方法内，继续执行
     12.1: 执行a方法的增强逻辑-> 事务增强器 (环绕增强-后置增强逻辑)
     12.2: 提交事务/回滚事务
     12.3: 恢复连接状态 （将conn的autocommit 设置回 true...等等）
     12.4: 清理工作（将绑定的conn资源从TransactionSync...Manager#resources移除）
     12.5: conn 连接关闭 （归还连接到datasource）

## SUPPORTS

SUPPORTS:当前存在事务，则加入当前事务，如果当前没有事务，就以非事务方法执行。
 如果A中有事务，则B方法的事务加入A事务中，成为一个事务（一起成功，一起失败），如果A中没有事务，那么B就以非事务方式运行（执行完直接提交）。



```java
class ServiceA {
    @Autowired 
    ServiceB serviceB；
    
    @Transactional
    void a() {
        ....
        serviceB.b();
        ....
    }
}

class ServiceB {
    @Transactional(propagation = SUPPORTS)   
    void b() {
        ......
    }
}
```

逻辑和上面 完全一致。



```java
class ServiceA {
    @Transactional(Propagation = SUPPORTS)
    void a() {
        ....

        ....
    }
}
```

线程在未绑定事务的情况下，去调用serviceA.a() 方法会发生什么呢？

1. 线程执行到serviceA.a() 方法时，其实是执行的  代理serviceA对象的a方法。
2. 执行代理serviceA对象的a方法
    2.1: 执行a方法的增强逻辑-> 事务增强器 (环绕增强)
    2.2: 事务增强器会做什么事? 提取事务标签属性
    2.3: 检查当前线程有没有绑定 conn 数据库连接 资源？ 发现当前线程未绑定（TransactionSync...Manager#resources 是 ThreadLocal<Map<obj,obj>>，检查key:datasource 有没有数据）
    2.4: 啥也不用做..
3. 执行事务增强器后面的增强器..
4. 最后一个advice调用 target的目标方法 a() 方法
    4.1: 假设target a方法 需要访问数据库 执行SQL 的话，程序需要获取一个 conn 资源，到哪拿？ DataSourceUtils.getConnection(datasource) ，因为事务增强器前置增强逻辑 并没有 向TransactionSync..Manager#resources 内绑定conn资源
    4.2: 因为 上一步未拿到 conn资源，所以 DataSourceUtils 通过 datasource.getConnection() 获取了一个全新的 conn 资源（注意：conn.autocommit == true,执行的每一条sql 都是一个 独立事务！！）
    4.3: 执行方法a逻辑...可能会执行一些 SQL 语句...
5. 线程继续执行到代理serviceA对象的a方法 （事务增强器-后置增强逻辑）
    5.1: 检查发现 TrasactionSync..Manager#resources 并未绑定任何 conn 资源，所以 这一步啥也不做了...

## MANDATORY

MANDATORY（mandatory：强制性的）:当前存在事务，则加入当前事务，如果当前事务不存在，则抛出异常。
 如果A中有事务，则B方法的事务加入A事务中，成为一个事务（一起成功，一起失败）；如果A中没有事务，B中有事务，那么B就直接抛异常了，意思是B必须要支持回滚的事务中运行。



```java
class ServiceA {
    @Autowired 
    ServiceB serviceB；
    
    @Transactional
    void a() {
        ....
        serviceB.b();
        ....
    }
}

class ServiceB {
    @Transactional(propagation = MANDATORY)
    void b() {
        ......
    }
}   
```

如果是这样的话，情况和 PROPAGATION_REQUIRED 案例分析 完全一致。



```java
class ServiceA {
    @Transactional(Propagation = MANDATORY)
    void a() {
        ....

        ....
    }
}
```

线程在未绑定事务的情况下，去调用serviceA.a() 方法会发生什么呢？

1. 线程执行到serviceA.a() 方法时，其实是执行的  代理serviceA对象的a方法。
2. 执行代理serviceA对象的a方法
    2.1: 执行a方法的增强逻辑-> 事务增强器 (环绕增强)
    2.2: 事务增强器会做什么事? 提取事务标签属性
    2.3: 检查当前线程有没有绑定 conn 数据库连接 资源？ 发现当前线程未绑定（TransactionSync...Manager#resources 是 ThreadLocal<Map<obj,obj>>，检查key:datasource 有没有数据）
    2.4: 直接抛出异常...

## REQUIRES_NEW

REQUIRES_NEW:创建一个新事务，如果存在当前事务，则挂起该事务。
 B会新建一个事务，A和B事务互不干扰，他们出现问题回滚的时候，也都只回滚自己的事务；A方法调用B方法；不管A方法有没有事务，B方法都新建一个自己的事务。



```java
class ServiceA {
    @Autowired 
    ServiceB serviceB；
    
    @Transactional
    void a() {
        ....
        serviceB.b();
        ....
    }
}

class ServiceB {
    @Transactional(propagation = REQUIRES_NEW)
    void b() {
        ......
    }
}
```

1. 线程执行到serviceA.a() 方法时，其实是执行的  代理serviceA对象的a方法。
2. 执行代理serviceA对象的a方法
    2.1: 执行a方法的增强逻辑-> 事务增强器 (环绕增强)
    2.2: 事务增强器会做什么事? 提取事务标签属性
    2.3: 检查当前线程有没有绑定 conn 数据库连接 资源？ 发现当前线程未绑定（TransactionSync...Manager#resources 是 ThreadLocal<Map<obj,obj>>，检查key:datasource 有没有数据）
    2.4: 因为未绑定conn资源，所以线程下一步就是 到 datasource.getConnection() 获取一个conn资源
    2.5: 因为新获取的conn资源的autocommit是true，所以这一步 修改 autocommit 为false，表示手动提交事务，这一步也表示 开启事务（修改conn其它 属性..）
    2.6: 绑定conn资源到 TransactionSync...Manager#resources，key：datasource
3. 执行事务增强器后面的增强器..
4. 最后一个advice调用 target的目标方法 a() 方法
    4.1: 假设target a方法 需要访问数据库 执行SQL 的话，程序需要获取一个 conn 资源，到哪拿？ DataSourceUtils.getConnection(datasource) 这一步最终会拿到 事务增强器 前置增强逻辑 存放在 TransactionSync..Manager#resources 内的
    conn 资源
    4.2: 执行方法a逻辑...可能会执行一些 SQL 语句...
5. 线程执行到这样一行代码：serviceB.b()
6. serviceB 它是一个代理对象，因为它也使用了 @Transactional 注解了，Spring 会为它创建代理的。
7. 执行代理serviceB对象的b方法
    7.1: 执行b方法的增强逻辑-> 事务增强器（环绕增强）
    7.2: 事务增强器会做什么事? 提取事务标签属性
    7.3: 检查发现当前线程已经绑定了conn资源（并且手动开启了事务..），又发现 当前方法的 传播行为：REQUIRES_NEW ，需要开启一个新的事务..
    7.4: 将已经绑定的conn资源 保存到 suspand 变量内
    7.5: 因为 REQUIRES_NEW 不会和上层共享同一个事务，所以这一步 又到 datasource.getConnection() 获取了一个全新的 conn 数据库连接资源
    7.6: 因为新获取的conn资源的autocommit是true，所以这一步 修改 autocommit 为false，表示手动提交事务，这一步也表示 开启事务（修改conn其它 属性..）
    7.7: 绑定conn资源到 TransactionSync...Manager#resources，key：datasource
8. 执行事务增强器后面的增强器..
9. 最后一个advice调用 target （serviceB）的目标方法 b() 方法
    9.1: 假设target b方法 需要访问数据库 执行SQL 的话，程序需要获取一个 conn 资源，到哪拿？ DataSourceUtils.getConnection(datasource) 这一步最终会拿到 事务增强器 前置增强逻辑 存放在 TransactionSync..Manager#resources 内的
    conn 资源
    9.2: 执行方法a逻辑...可能会执行一些 SQL 语句...
10. 线程继续执行 事务增强器 环绕增强的后置逻辑 （代理serviceB.b() 方法的 后置增强）
     10.1: 检查发现，serviceB.b() 事务是 b方法开启的，所以 需要做一些事情了
     10.1: 执行b方法的增强逻辑-> 事务增强器 (环绕增强-后置增强逻辑)
     10.2: 提交事务/回滚事务
     10.3: 恢复连接状态 （将conn的autocommit 设置回 true...等等）
     10.4: 清理工作（将绑定的conn资源从TransactionSync...Manager#resources移除）
     10.5: conn 连接关闭 （归还连接到datasource）
     10.6: 检查suspand 发现 该变量有值，需要执行 恢复现场的工作 resume()
11. 恢复现场
     11.1: 将suspand 挂起的 conn 资源再次  绑定到 TransactionSync...Manager#resources 内，方便 serviceA 继续使用它的conn资源 （它自己的事务）
12. 线程继续回到 serviceA.a() 方法内
     12.1: 继续执行一些sql ...注意 这里它使用的 conn 是 serviceA 申请的 conn
13. 线程继续执行 事务增强器 环绕增强的后置逻辑 （代理serviceA.a() 方法的 后置增强）
     10.1: 检查发现，serviceA.a() 事务是 a方法开启的，所以 需要做一些事情了
     10.1: 执行a方法的增强逻辑-> 事务增强器 (环绕增强-后置增强逻辑)
     10.2: 提交事务/回滚事务
     10.3: 恢复连接状态 （将conn的autocommit 设置回 true...等等）
     10.4: 清理工作（将绑定的conn资源从TransactionSync...Manager#resources移除）
     10.5: conn 连接关闭 （归还连接到datasource）

## NOT_SUPPORTED

NOT_SUPPORTED: 以非事务方式执行,如果当前存在事务，则挂起当前事务
 被调用者B会以非事务方式运行（直接提交），如果当前有事务，也就是A中有事务，A会被挂起（不执行，等待B执行完，返回）；A和B出现异常需要回滚，互不影响

## NEVER

NEVER: 如果当前没有事务存在，就以非事务方式执行；如果有，就抛出异常。就是B从不以事务方式运行
 A中不能有事务，如果没有，B就以非事务方式执行，如果A存在事务，那么直接抛异常

## NESTED

NESTED： 嵌套事务:如果当前事务存在，则在嵌套事务中执行，否则REQUIRED的操作一样(开启一个事务)
 如果A中没有事务，那么B创建一个事务执行，如果A中也有事务，那么B会会把事务嵌套在里面。