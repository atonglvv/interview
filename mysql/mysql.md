# 事务的四大特性

## 原子性（Atomicity）

要么全部成功，要么全部失败。

原子性，在 InnoDB 里面是通过 undo log 来实现的，它记录了数据修改之前的值（逻辑日志），一旦发生异常，就可以用 undo log 来实现回滚操作。

## 一致性（Consistent）

指的是数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态。比如主键必须是唯一的，字段长度符合要求。

除了数据库自身的完整性约束，还有一个是用户自定义的完整性。

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

对于MySQL来说，假设T1先根据搜索条件P读取了一些记录，接着T2删除了一些符合搜索条件P的记录后提交，如果T1再读取符合相同搜索条件的记录时获得了不同的结果集，我们就可以把这种现象认为是结果集中的每一条记录分别发生了不可重复读现象。

## 幻读（Phantom）

如果一个事务先根据某些搜索条件查询出一些记录，在该事物未提交时，另一个事务写入了一些符合那些搜索条件的记录（这里的写入可以指INSERT、DELETE、UPDATE操作），就意味着发生了幻读。

对于MySQL来说，幻读强调的就是一个事务在按照某个相同的搜索条件多次读取记录时，在后读取时读到了之前没有读到的记录，这个“后续取到的之前没有读到的记录”可以是由别的事务执行INSERT语句插入的，也可能是别的事务执行了更新记录键值的UPDATE语句而插入的。

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

在LBCC中，**读写冲突**，会使用诸如记录锁、间隙锁与临键锁等锁来实现数据的并发安全，因此读写性能不高。

`LBCC`是`Lock-Based Concurrent Control`的简称，意思是基于锁的并发控制。在`InnoDB`中按锁的模式来分的话可以分为共享锁（S）、排它锁（X）和意向锁，其中意向锁又分为意向共享锁（IS）和意向排它锁（IX）；如果按照锁的算法来分的话又分为记录锁（`Record Locks`）、间隙锁（`Gap Locks`）和临键锁（`Next-key Locks`）。其中临键锁就可以用来解决RR下的幻读问题。

### 记录锁（Record Locks）

第一种情况，当我们对于唯一性的索引（包括唯一索引和主键索引）使用等值查询，精准匹配到一

条记录的时候，这个时候使用的就是记录锁。比如 where id = 1 。

对表中的行记录加锁，叫做记录锁，简称行锁。可以使用`sql`语句`select ... for update`来开启锁，`select`语句**必须为精准匹配**（=），不能为范围匹配，且**匹配列字段必须为唯一索引或者主键列**。也可以通过对查询条件为主键索引或唯一索引的数据行进行`UPDATE`操作来添加记录锁。

### 间隙锁（GAP Locks）

第二种情况，当我们查询的记录不存在，无论是用等值查询还是范围查询的时候，它使用的都是间隙锁。

间隙锁是对范围加锁，但不包括已存在的索引项。可以使用`sql`语句`select ... for update`来开启锁，`select`语句为**范围查询**，匹配列字段为索引项，且没有数据返回；或者`select`语句为等值查询，匹配字段为唯一索引，也没有数据返回。

间隙锁有一个比较致命的弱点，就是当锁定一个范围键值之后，即使某些不存在的键值也会被无辜的锁定，而造成在锁定的时候无法插入锁定键值范围内的任何数据。在某些场景下这可能会对性能造成很大的危害。以下是加锁之后，插入操作的例子：

```sql
select * from user where id > 15 for update;
//插入失败，因为id20大于15，不难理解
insert into user values(20,'20');
//插入失败，原因是间隙锁锁的是记录间隙，而不是sql，也就是说`select`语句的锁范围是（11，+∞），而13在这个区间中，所以也失败。
insert into user values(13,'13');
```

`GAP Locks`只存在于RR隔离级别下，它锁住的是间隙内的数据。加完锁之后，间隙中无法插入其他记录，并且锁的是记录间隙，而非`sql`语句。间隙锁之间都不存在冲突关系。

**打开间隙锁设置：** 以通过命令`show variables like 'innodb_locks_unsafe_for_binlog';`来查看 `innodb_locks_unsafe_for_binlog` 是否禁用。`innodb_locks_unsafe_for_binlog`默认值为OFF，即启用间隙锁。因为此参数是只读模式，如果想要禁用间隙锁，需要修改 `my.cnf`（windows是`my.ini`） 重新启动才行。

```shell
#在 my.cnf 里面的[mysqld]添加
[mysqld]
innodb_locks_unsafe_for_binlog = 1
```

### 临键锁（Next-Key Locks）

第三种情况，当我们使用了范围查询，不仅仅命中了 Record 记录，还包含了 Gap 间隙，在这种情况下我们使用的就是临键锁，它是 MySQL 里面默认的行锁算法，相当于记录锁加上间隙锁。

比如我们使用>5 <9 ， 它包含了不存在的区间，也包含了一个 Record 7。

锁住最后一个 key 的下一个左开右闭的区间。

select * from t2 where id >5 and id <=7 for update; 锁住(4,7]和(7,10]

select * from t2 where id >8 and id <=10 for update; 锁住 (7,10]，(10,+∞)

总结：为什么要锁住下一个左开右闭的区间？——就是为了解决幻读的问题。

当我们对上面的记录和间隙共同加锁时，添加的便是临键锁（左开右闭的集合加锁）。为了防止幻读，临键锁阻止特定条件的新记录的插入，因为插入时要获取插入意向锁，与已持有的临键锁冲突。可以使用`sql`语句`select ... for update`来开启锁，`select`语句为范围查询，匹配列字段为索引项，且有数据返回；或者`select`语句为等值查询，匹配列字段为索引项，不管有没有数据返回。

插入意向锁并非意向锁，而是一种特殊的间隙锁。

### 总结

- 如果查询没有命中索引，则退化为表锁;
- 如果等值查询唯一索引且命中唯一一条记录，则退化为行锁;
- 如果等值查询唯一索引且没有命中记录，则退化为临近结点的间隙锁;
- 如果等值查询非唯一索引且没有命中记录，退化为临近结点的间隙锁(包括结点也被锁定)；如果命中记录，则锁定所有命中行的临键锁，并同时锁定最大记录行下一个区间的间隙锁。
- 如果范围查询唯一索引或查询非唯一索引且命中记录，则锁定所有命中行的临键锁 ，并同时锁定最大记录行下一个区间的间隙锁。
- 如果范围查询索引且没有命中记录，退化为临近结点的间隙锁(包括结点也被锁定)。

### 当前读

当前读（`Locking Read`）也称锁定读，读取当前数据的最新版本，而且读取到这个数据之后会对这个数据加锁，防止别的事务更改即通过`next-key`锁（行锁+gap锁）来解决当前读的问题。在进行写操作的时候就需要进行“当前读”，读取数据记录的最新版本，包含以下`SQL`类型：`select ... lock in share mode` 、`select ... for update`、`update` 、`delete` 、`insert`。

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

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1463/1650529519068/fd9bdf3f70114287b8f81df5c54c1f32.png)

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

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1463/1650529519068/772aa82513ed4458bdce19ce2cbfe2ec.png)

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









