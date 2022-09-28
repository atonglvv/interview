# buffer pool

InnoDB 引擎存储数据的时候，是以页为单位的，每个数据页的大小默认是 16KB，可以通过如下命令来查看页的大小：

​	![](\img\innodb_page_size.png)

16384/1024=16，刚好是 16KB。

计算机在存储数据的时候，最小存储单元是扇区，一个扇区的大小是 512 字节，而文件系统（例如 XFS/EXT4）最小单元是块，一个块的大小是 4KB，也就是四个块组成一个 InnoDB 中的页。

如果每一次操作都操作磁盘，那么就会产生海量的磁盘 IO 操作，如果是传统的机械硬盘，还会涉及到很多随机 IO 操作，效率低的令人发指。这严重影响了 MySQL 的性能。

为了解决这一问题，MySQL 引入了 buffer pool，也就是我们常说的缓冲池。

buffer pool 的主要作用就是缓存索引和表数据，以避免每一次操作都要进行磁盘 IO，通过 buffer pool 可以提高数据的访问速度。

# change buffer

buffer pool 虽然提高了访问速度，但是增删改的效率并没有因此提升，当涉及到增删改的时候，还是需要磁盘 IO，那么效率一样低。

为了解决这个问题，MySQL 中引入了 change buffer。change buffer 以前并不叫这个名字，以前叫 insert buffer，即只针对 insert 操作有效，现在改名叫 

change buffer 了，不仅仅针对 insert 有效，对 delete 和 update 操作也是有效的，change buffer 主要是对非唯一的索引有效，如果字段是唯一性索引，那么更

新的时候要去检查唯一性，依然无法避免磁盘 IO。

change buffer 就是说，当我们需要更改数据库中的数据的时候，我们把更改记录到内存中，等到将来数据被读取的时候，再将内存中的数据 merge 到 buffer pool 然后返回，此时 buffer pool 中的数据和磁盘中的数据就会有差异，有差异的数据我们称之为脏页，在满足条件的时候（redo log 写满了、内存写满了、其他空闲时候），InnoDB 会把脏页刷新回磁盘。这种方式可以有效降低写操作的磁盘 IO，提升数据库的性能。

通过如下命令我们可以查看 change buffer 的大小以及哪些操作会涉及到 change buffer：

​	![](\..\img\change_buffer.png)

- innodb_change_buffer_max_size：这个配置表示 change buffer 的大小占整个缓冲池的比例，默认值是 `25%`，最大值是 `50%`。
- innodb_change_buffering：这个操作表示哪些写操作会用到 change buffer，默认的 all 表示所有写操作，我们也可以自己设置为 `none`/`inserts`/`deletes`/`changes`/`purges` 等。

不过 change buffer 和 buffer pool 都涉及到内存操作，数据不能持久化，那么，当存在脏页的时候，MySQL 如果突然挂了，就有可能造成数据丢失（因为内存中的数据还没写到磁盘上），但是我们在实际使用 MySQL 的时候，其实并不会有这个问题，那么问题是怎么解决的？那就得靠 redo log 了。

# WAL

WAL: Write-Ahead Logging  预写日志系统

其主要是指MySQL在执行写操作的时候并不是立刻更新到磁盘上，而是先记录在日志中，之后在合适的时间更新到磁盘中。日志主要分为undo log、redo log、binlog。

MySQL真正使用WAL的原因是：磁盘的写操作是随机IO，比较耗性能，所以如果把每一次的更新操作都先写入log中，那么就成了顺序写操作，实际更新操作由后台线程再根据log异步写入。这样对于client端，延迟就降低了。并且，由于顺序写入大概率是在一个磁盘块内，这样产生的IO次数也大大降低。所以WAL的核心在于将随机写转变为了顺序写，降低了客户端的延迟，提升了吞吐量。

# redo log

MYSQL中事务的原子性和持久性离不开Redo Log。

**Redo Log 是存储引擎层（Innodb）的。binlog是Server层的。**

考虑：`mysql`是如何保证持久性的呢？

最简单的做法是在每次事务提交的时候，将该事务涉及修改的数据页全部刷新到磁盘中。但是这么做会有严重的性能问题，主要体现在两个方面：

- 因为`Innodb`是以页为单位进行磁盘交互的，而一个事务很可能只修改一个数据页里面的几个字节，这个时候将完整的数据页刷到磁盘的话，浪费资源！
- 一个事务可能涉及修改多个数据页，并且这些数据页在物理上并不连续，使用随机IO写入性能太差！

咋办呢？我们的初衷是让已经提交了的事务对数据库中的数据所做的修改能永久生效，及时后面系统崩溃，在重启后也能把这种修改恢复过来。其实，没有必要在每次提交事务时就把该事务在内存中修改过的全部页面刷新到磁盘，只需要把修改的内容记录一下就好。这就是`redo log`。

`mysql`设计了**redo log**来解决上面的性能问题。

# redo log 优点

- redo log 占用的空间非常小。
- redo log 是顺序写入磁盘。（顺序IO）

# redo log 写入时机

`redo log`包括两部分：一个是内存中的日志缓冲(`redo log buffer`)，另一个是磁盘上的日志文件(`redo log file`)。

`mysql`每执行一条`DML`语句，先将记录写入`redo log buffer`，后续某个时间点（**写入时机**）再一次性将多个操作记录写到`redo log file`。这种**先写日志，再写磁盘**的技术就是`MySQL`里经常说到的`WAL(Write-Ahead Logging)` 技术。



在计算机操作系统中，**用户空间**(`user space`)下的缓冲区数据一般情况下是无法直接写入磁盘的，中间必须经过操作系统**内核空间**(`kernel space`)缓冲区(`OS Buffer`)。因此，`redo log buffer`写入`redo log file`实际上是先写入`OS Buffer`，然后再通过系统调用`fsync()`将其刷到`redo log file`中，过程如下：

![fsync](\img\fsync.jpg)	



`mysql`支持三种将`redo log buffer`写入`redo log file`的时机，可以通过`innodb_flush_log_at_trx_commit`参数配置，各参数值含义如下：

| 参数值              | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| 0（延迟写）         | 事务提交时不会将`redo log buffer`中日志写入到`os buffer`，而是每秒写入`os buffer`并调用`fsync()`写入到`redo log file`中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，**当系统崩溃，会丢失1秒钟的数据**。 |
| 1（实时写，实时刷） | 事务每次提交都会将`redo log buffer`中的日志写入`os buffer`并调用`fsync()`刷到`redo log file`中。这种方式**即使系统崩溃也不会丢失任何数据**（如果事务执行期间，mysql宕机，因为事务没有提交，所以又不会有损失），但是因为每次提交都写入磁盘，IO的性能较差。 |
| 2（实时写，延迟刷） | 每次提交都仅写入到`os buffer`，然后是每秒调用`fsync()`将`os buffer`中的日志写入到`redo log file`。（不涉及`redo log buffer`） |

# redo log 的日志格式

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

# checkpoint

Checkpoint机制就是将脏数据（在内存中更新但没刷盘）刷新回磁盘的机制，即只要发生Checkpoint，就要将脏数据刷新回磁盘，反过来，当MySQL重启时会去找Checkpoint，并且根据Checkpoint的特性。MySQL可以明确的知道checkponit之前的脏数据已经落过盘了，重启时没必要进行重做。

Checkpoint机制的作用：

1、所谓的崩溃恢复，其实就是MySQL重启时照着redo log中的最后一次Checkpoint之后的日志回放一遍

2、因为Checkpoint会不断的更新，并且MySQL重启时只需要对Checkpoint之后的数据进行恢复，所以Checkpoint会缩短MySQL重启的时间。

3、因此每次进行Checkpoint时buffer pool中的脏数据页、redo log中的脏日志都会落盘。所以Checkpoint实际上起到了为这两者进行瘦身的作用。维持两个的可用性。

# MiNi-Transaction

以组的形式写入redo日志。

一个sql语句可能修改若干个页面（page），比如一条insert语句可能修改系统表空间页面的max row id属性，还会更新b+树聚簇索引和二级索引对应的页面。由于这些都是发生在buffer pool里，这时候都需要记录在redo日志里，于是在记录的时候，innoDB又把这些分为不同的组。

修改max row_id是不可分割的，向聚簇索引对应的b+树插入一条记录产生的redo日志不可分割，向某个二级索引b+树插入一条记录产生的redo日志不可分割，还有一些其他页面生成的redo日志不可分割。

怎么理解不可分割呢？我们向某个索引对应的b+树插入一记录，在插入b+树之前，需要定位向哪个叶子节点代表的数据页，定位到具体的数据页后，有两种可能：

- 情况一：该数据页的剩余空闲空间充足，足够容纳这一条待插记录，这种就很简单，直接把记录插入这个数据页，记录一条MLOG_COMP_REC_INSERT记录到redo日志，我们吧这种情况称为乐观插入。

- 情况二：该数据页的剩余空间不足，那么这时候就需要进行所谓的页分裂操作，也就是新建一个叶子节点，插入一条10的记录，那么比10大的记录都会移到新建的叶子节点，把10插到前面的叶子节点，如果内节点不够用，则同样也需要分裂，这样会记录更多的redo日志，我们称这种为悲观插入。对于这些，我们还需要修改各种段，区的统计信息，各种链表的统计信息，（比如free链表，fsp_free_page链表等等），反正有二三十条记录。

这些操作肯定必须是原子性的，比如不能在系统宕机的时候，redo日志吧聚簇索引的修改记录恢复，而二级索引的修改记录未恢复，这种必定导致数据错误，形成不完成的b+树。如何保证原子性呢，于是innoDB规定这些操作必须以【组】的形式来记录到redo日志，在系统宕机重启时候，要么全部一起恢复，要么一条都不恢复。那么怎么做到呢？

当悲观插入，需要记录多条redo日志。innoDB解决办法是，每次一组日志之后，记录一条特殊的redo日志，MLOG_MULTI_REC_END,type字段对应的十进制数字为31，该类型的结构很简单，只有一个type字段。所以当系统崩溃时候，只有解析到MLOG_MULTI_REC_END才会算是一组完整的操作，如果没有解析到MLOG_MULTI_REC_END，则之前解析的都放弃。

如果有的操作只记录一条数据，然后海特意记录一条MLOG_MULTI_REC_END，不是很浪费吗，这时候如果type字段的第一个比特位是1，代表需要该原子性操作只需要操作一条redo日志。

## MTR

对底层页面进行一次原子访问的过程称为一个 Mini-Transaction（MTR）。

![](\img\mtr.png)	

# redo log 的记录形式

`redo log`实际上记录数据页的变更，而这种变更记录是没必要全部保存，因此`redo log`实现上采用了大小固定，循环写入的方式，当写到结尾时，会回到开头循环写日志。如下图：

![](\img\redo_log_file.jpg)	

**在innodb中，既有`redo log`需要刷盘，还有`数据页`也需要刷盘，`redo log`存在的意义主要就是降低对`数据页`刷盘的要求**。在上图中，`write pos`表示`redo log`当前记录的`LSN`(逻辑序列号)位置，`check point`表示**数据页更改记录**刷盘后对应`redo log`所处的`LSN`(逻辑序列号)位置。`write pos`到`check point`之间的部分是`redo log`空着的部分，用于记录新的记录；`check point`到`write pos`之间是`redo log`待落盘的数据页更改记录。当`write pos`追上`check point`时，会先推动`check point`向前移动，空出位置再记录新的日志。

启动`innodb`的时候，不管上次是正常关闭还是异常关闭，总是会进行恢复操作。因为`redo log`记录的是数据页的物理变化，因此恢复的时候速度比逻辑日志(如`binlog`)要快很多。 重启`innodb`时，首先会检查磁盘中数据页的`LSN`，如果数据页的`LSN`小于日志中的`LSN`，则会从`checkpoint`开始恢复。 还有一种情况，在宕机前正处于`checkpoint`的刷盘过程，且数据页的刷盘进度超过了日志页的刷盘进度，此时会出现数据页中记录的`LSN`大于日志中的`LSN`，这时超出日志进度的部分将不会重做，因为这本身就表示已经做过的事情，无需再重做。

# 崩溃恢复

在服务器不宕机的情况下，redo log简直就是个累赘，不仅没用，反而让性能变得更差。但是，万一数据库挂了，我们就可以在重启时根据redo log将页面恢复到系统崩溃前的状态。







































