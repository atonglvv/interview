# 事务的四大特性

## 原子性（Atomicity）

要么全部成功，要么全部失败。

原子性，在 InnoDB 里面是通过 undo log 来实现的，它记录了数据修改之前的值（逻辑日志），一旦发生异常，就可以用 undo log 来实现回滚操作。

## 一致性（Consistent）

指的是数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态。比如主键必须是唯一的，字段长度符合要求。

除了数据库自身的完整性约束，还有一个是用户自定义的完整性。

数据库某些操作的原子性和隔离性都是保证一致性的一种手段，在操作执行完成后保证符合所有既定的约束则是一种结果。那么，满足原子性和隔离性的操作就一定能满足一致性么？这倒也不一定。比如，狗哥要转账20元给猫爷，虽然这满足原子性和隔离性，但是在转账完成后狗哥账户的余额就成负的了，这显然不满足一致性。那么，不满足原子性和隔离性的操作就一定不满足一致性么？也不一定，只要最后的结果符合所有现实世界中的约束，那么就符合一致性。

## 隔离性（isolation）

很多个的事务，对表或者 行的并发操作，应该是透明的，互相不干扰的。通过这种方式，我们最终也是保证业务数据的一致性。

## 持久性（durable）

redo log 实现。

# 事务并发执行时遇到的一致性问题

## 脏写（Dirty Write）

如果一个事务修改了另一个未提交事务修改过的数据，就意味着发生了脏写。

## 脏读（Dirty Read）

如果一个事务读到了另一个未提交事务修改过的数据，就意味着发生了脏读。

## 不可重复读（Non-Repeatable Read）

如果一个事务修改了另一个未提交事务读取的数据，就意味着发生了不可重复读。

## 幻读（Phantom）

如果一个事务T1先根据某些搜索条件查询出一些记录，在该事物未提交时，另一个事务T2写入了一些符合那些搜索条件的记录（这里的写入可以指INSERT、DELETE、UPDATE操作），就意味着发生了幻读。

对于MySQL来说，幻读强调的就是一个事务在按照某个相同的搜索条件多次读取记录时，在后读取时读到了之前没有读到的记录，这个“后续取到的之前没有读到的记录”可以是由别的事务执行INSERT语句插入的，也可能是别的事务执行了更新记录键值的UPDATE语句而插入的。

假设T1先根据搜索条件P读取了一些记录，接着T2删除了一些符合搜索条件P的记录后提交，如果T1再读取符合相同搜索条件的记录时获得了不同的结果集，我们就可以把这种现象认为是结果集中的每一条记录分别发生了**不可重复读**现象。

# 隔离级别（SQL92标准）

## Read Uncommitted（未提交读）

一个事务可以读取到其他事务未提交的数据，会出现脏读，所以叫做 RU，它没有解决任何的问题。

## Read Committed（已提交读）

一个事务只能读取到其他事务已提交的数据，不能读取到其他事务未提交的数据，它解决了脏读的问题，但是会出现不可重复读的问题。

## Repeatable Read（可重复读）

它解决了不可重复读的问题，也就是在同一个事务里面多次读取同样的数据结果是一样的，但是在这个级别下，没有定义解决幻读的问题。

注意：mysql innodb RR隔离级别下很大程度解决了幻读问题。

## Serializable（串行化）

在这个隔离级别里面，所有的事务都是串行执行的，也就是对数据的操作需要排队，已经不存在事务的并发操作了，所以它解决了所有的问题。

# innodb如何保证隔离？

## LBCC

具体请参考同级目录下 lbcc.md。

## MVCC

在MVCC中，**读写不冲突**，记录每一行的多个版本，来避免在多个事务之间的竞争。以空间换时间的思路，极大地提高了读写性能。

MVCC主要靠undo log版本链与ReadView来实现。

具体请参考同级目录下 mvcc.md。



# 慢sql

[`https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html`](https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html)

## 打开慢日志开关

因为开启慢查询日志是有代价的（跟 binlog、optimizer-trace 一样），所以它默认是关闭的：

```sql
show variables like 'slow_query%';
```

![](img\慢sql01.png)	

除了这个开关，还有一个参数，控制执行超过多长时间的 SQL 才记录到慢日志，默认是 10 秒。

```sql
show variables like '%long_query%';
可以直接动态修改参数（重启后失效）。
set @@global.slow_query_log=1; -- 1 开启，0 关闭，重启后失效 
set @@global.long_query_time=3; -- mysql 默认的慢查询时间是 10 秒，另开一个窗口后才会查到最新值 

show variables like '%long_query%'; 
show variables like '%slow_query%';
或者修改配置文件 my.cnf。
以下配置定义了慢查询日志的开关、慢查询的时间、日志文件的存放路径。
slow_query_log = ON 
long_query_time=2 
slow_query_log_file =/var/lib/mysql/localhost-slow.log
模拟慢查询：
select sleep(10);
查询 user_innodb 表的 500 万数据（检查是不是没有索引）。
SELECT * FROM `user_innodb` where phone = '136';
```

##### **4.1.2 慢日志分析**

**1、日志内容**

```sql
show global status like 'slow_queries'; -- 查看有多少慢查询 
show variables like '%slow_query%'; -- 获取慢日志目录
cat /var/lib/mysql/ localhost-slow.log
```

![](img\慢sql02.png)	

```
有了慢查询日志，怎么去分析统计呢？比如 SQL 语句的出现的慢查询次数最多，平均每次执行了多久？人工肉眼分析显然不可能。
```

**2、mysqldumpslow**

```
https://dev.mysql.com/doc/refman/5.7/en/mysqldumpslow.html
```

MySQL 提供了 mysqldumpslow 的工具，在 MySQL 的 bin 目录下。

```sql
mysqldumpslow --help
```

例如：查询用时最多的 10 条慢 SQL：

```sql
mysqldumpslow -s t -t 10 -g 'select' /var/lib/mysql/localhost-slow.log
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1463/1650529519068/2c2a6c2ff2154c99a7ca6362b2c34615.png)

Count 代表这个 SQL 执行了多少次；

Time 代表执行的时间，括号里面是累计时间；

Lock 表示锁定的时间，括号是累计；

Rows 表示返回的记录数，括号是累计。

除了慢查询日志之外，还有一个 SHOW PROFILE 工具可以使用

# 数据库持久化原理

redo log来保证数据的持久化。

# 事物的回滚怎么实现？

undo log 用于事务的回滚跟mvcc

# redo log，undo log是所有存储引擎都有的么？

redo log，undo log 是属于innodb，引擎层面的。



# mysql事务的原子性，一致性，持久性，隔离性是怎么实现的？

对于Innodb它主要由俩个事务日志文件redoLog和undoLog来保证事务的原子性，一致性，持久性；隔离性由锁来控制 如间隙锁，排它锁。



# Lock

## 写入数据库的时候会有什么锁？





# 主从复制架构

## 优点

- 数据热备份，高可用
- 可以做读写分离，实现高并发





# 一张表最多可以建多少个索引？

16个。





# 什么是索引下推（ICP）？









