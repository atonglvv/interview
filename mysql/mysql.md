# 数据库持久化原理

redo log来保证数据的持久化。

# 事物的回滚怎么实现？

undo log 用于事务的回滚跟mvcc

# redo log，undo log是所有存储引擎都有的么？

redo log，undo log 是属于innodb，引擎层面的。



# mysql事务的原子性，一致性，持久性，隔离性是怎么实现的？

对于Innodb它主要由俩个事务日志文件redoLog和undoLog来保证事务的原子性，一致性，持久性；隔离性由锁来控制 如间隙锁，排它锁。



# redo log

考虑：`mysql`是如何保证持久性的呢？

最简单的做法是在每次事务提交的时候，将该事务涉及修改的数据页全部刷新到磁盘中。但是这么做会有严重的性能问题，主要体现在两个方面：

- 因为`Innodb`是以页为单位进行磁盘交互的，而一个事务很可能只修改一个数据页里面的几个字节，这个时候将完整的数据页刷到磁盘的话，浪费资源！
- 一个事务可能涉及修改多个数据页，并且这些数据页在物理上并不连续，使用随机IO写入性能太差！

咋办呢？我们的初衷是让已经提交了的事务对数据库中的数据所做的修改能永久生效，及时后面系统崩溃，在重启后也能把这种修改恢复过来。其实，没有必要在每次提交事务时就把该事务在内存中修改过的全部页面刷新到磁盘，只需要把修改的内容记录一下就好。这就是`redo log`。

`mysql`设计了**redo log**来解决上面的性能问题。

## redo log 优点

- redo log 占用的空间非常小。
- redo log 是顺序写入磁盘。（顺序IO）

## redo log 写入时机

`redo log`包括两部分：一个是内存中的日志缓冲(`redo log buffer`)，另一个是磁盘上的日志文件(`redo log file`)。`mysql`每执行一条`DML`语句，先将记录写入`redo log buffer`，后续某个时间点（**写入时机**）再一次性将多个操作记录写到`redo log file`。这种**先写日志，再写磁盘**的技术就是`MySQL`里经常说到的`WAL(Write-Ahead Logging)` 技术。



在计算机操作系统中，**用户空间**(`user space`)下的缓冲区数据一般情况下是无法直接写入磁盘的，中间必须经过操作系统**内核空间**(`kernel space`)缓冲区(`OS Buffer`)。因此，`redo log buffer`写入`redo log file`实际上是先写入`OS Buffer`，然后再通过系统调用`fsync()`将其刷到`redo log file`中，过程如下：



![fsync](img\fsync.jpg)



`mysql`支持三种将`redo log buffer`写入`redo log file`的时机，可以通过`innodb_flush_log_at_trx_commit`参数配置，各参数值含义如下：

| 参数值              | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| 0（延迟写）         | 事务提交时不会将`redo log buffer`中日志写入到`os buffer`，而是每秒写入`os buffer`并调用`fsync()`写入到`redo log file`中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，**当系统崩溃，会丢失1秒钟的数据**。 |
| 1（实时写，实时刷） | 事务每次提交都会将`redo log buffer`中的日志写入`os buffer`并调用`fsync()`刷到`redo log file`中。这种方式**即使系统崩溃也不会丢失任何数据**（如果事务执行期间，mysql宕机，因为事务没有提交，所以又不会有损失），但是因为每次提交都写入磁盘，IO的性能较差。 |
| 2（实时写，延迟刷） | 每次提交都仅写入到`os buffer`，然后是每秒调用`fsync()`将`os buffer`中的日志写入到`redo log file`。（不涉及`redo log buffer`） |



## redo log 的日志格式

**redo log** 只是记录了一下事务对数据库进行了哪些修改。

比如某个事务将系统表空间第100号页中偏移量为1000处的那个字节的值从1改成2，只需要进行如下记录：

>将第0号表空间第100号页中偏移量为1000处的值更新为2

**Innodb**定义了多种类型的redo log。但绝大部分类型的redo log 格式如下：

| type | space ID | page number | data |
| ---- | -------- | ----------- | ---- |

**type：**这条redo log的类型，在MySQL5.7.22版本中，Innodb一共设计了53种不同的类型。

**space ID：**表空间ID。

**page number：** 页号。

**data：**这条redo日志的具体内容。

## redo log 的记录形式

https://juejin.cn/post/6860252224930070536#heading-4



# Lock

## 写入数据库的时候会有什么锁？





# 主从复制架构

## 优点

- 数据热备份，高可用
- 可以做读写分离，实现高并发





# 一张表最多可以建多少个索引？

16个。









