# 文件目录布局

不考虑多副本的情况， 一个分区对应一个日志（ Log）。为了防止 Log 过大，Kafka 又引入了日志分段（ LogSegment ）的概念， 将 Log 切分为多个 LogSegment，相当于一个巨型文件被平均分配为多个相对较小的文件，这样也便于消息的维护和清理。

事实上， Log 和LogSegment也不是纯粹物理意义上的概念， **Log 在物理上只以文件夹的形式存储**，而**每个LogSegment 对应于磁盘上的一个日志文件和两个索引文件，以及可能的其他文件（比如以“ .txnindex ”为后缀的事务索引文件）**。 

Log对应了一个命名形式为<topic>-<partition>的文件夹。举个例子，假设有一个名为"topic-log"的topic，此topic中有4个分区，那么在实际物理存储上表现为"topic-log-0" "topic-log-1" "topic-log-2" "topic-log-3" 这4个文件夹。

![](.\img\日志.png)	  

向 Log 中追加消息时是顺序写入的，只有最后一个 LogSegment 才能执行写入操作，在此之前所有的 LogSegment 都不能写入数据。

为了便于消息的检索，每个 LogSegment 中的日志文件（以“ .log”为文件后缀）都有对应的两个索引文件 ：偏移量索引文件（以“ .index”为文件后缀）和时间戳索引文件（以“ .timeindex ”为文件后缀）。

每个 LogSegment 都有一个基准偏移量 baseOffset，用来表示当前 LogSegment中第一条消息的 offset 。 偏移量是一个 64 位的长整型数，日志文件和两个索引文件都是根据基准偏移量（ baseOffset）命名的，名称固定为 20 位数字，没有达到的位数则用 0 填充 。 比如第一个 LogSegment 的基准偏移量为 0，对应的日志文件为 00000000000000000000 .logo

举例说明,向主题 topic-log中发送一定量的消息,某一时刻 topic-log-0目录中的布局如下所示。

![](.\img\文件目录布局01.png)

注意每个LogSegment中不只包含".log" ".index" ".timeindex" 这三种文件，还可能包含".deleted" ".cleaned" ".swap" 等临时文件，以及可能的".snapshot" ".txnindex" "leader-epoch-checkpoint" 等文件。

从更加宏观的视角上看, Kafka中的文件不只上面提及的这些文件,比如还有一些检查点文件,当一个Kafka服务第一次启动的时候,默认的根目录下就会创建以下5个文件:

![](.\img\文件目录布局02.png)	

我们了解到消费者提交的位移是保存在Kafka内部的主题_consumer__offsets中的,初始情况下这个主题并不存在,当第一次有消费者消费消息时会自动创建这个主题在某一时刻,Kafka中的文件目录布局如图5-2所示。每一个根目录都会包含最基本的4个检查点文件(xxx-checkpoint)和 meta.properties文件。在创建主题的时候,如果当前 broker中不止配置了一个根目录,那么会挑选分区数最少的那个根目录来完成本次创建任务。

![](.\img\文件目录布局03.png)	

# 日志格式的演变

## 数据压缩

Kafka支持对消息进行压缩，以减少磁盘空间和网络带宽的使用。生产者可以在发送消息时选择压缩算法（如GZIP、Snappy或LZ4），然后将压缩后的消息批量发送到Kafka。Kafka会在不解压缩的情况下存储和传输压缩后的消息，从而提高性能。

消费者在读取压缩后的消息时，会自动解压缩并返回原始消息。这意味着消费者无需关心消息的压缩格式，可以透明地处理压缩和非压缩消息。

要启用数据压缩，您可以在生产者配置中设置**compression.type**选项，如下所示：

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("compression.type", "gzip"); // 设置压缩算法为GZIP
```

# 日志索引

偏移量索引文件用来建立消息偏移量（offset）到物理地址之间的映射关系，方便快速定位消息所在的物理文件位置；时间戳索引文件则根据指定的时间戳来查找对应的偏移量信息。

Kafka中的索引文件以稀疏索引的方式添加索引项，每当写入一定量（由broker端参数log.index.interval.bytes指定，默认为4k）的消息时，就会在偏移量索引文件和时间戳索引文件中增加一个索引项。

Kafka通过**MappedByteBuffer**将索引文件映射到内存中，来加快索引的查询速度。

## 偏移量索引

对于偏移量索引文件，保存的是 **<相对偏移量，物理地址>** 的对应关系，文件中的相对偏移量是单调递增的。

查询**指定偏移量**对应的消息时，使用**改进的二分查找算法**来快速定位偏移量的位置，如果指定的偏移量不在索引文件中，则会返回文件中小于指定偏移量的最大偏移量及对应的物理地址，该逻辑通过OffsetIndex.lookup()方法实现。

偏移量索引文件的索引项结构如下图所示，每个索引项记录了相对偏移量relativeOffset和对应消息的第一个字节在日志段文件中的物理地址position，共占用8个字节。

![](C:\Document\Github\interview\MQ\Kafka\img\偏移量索引项.png)	

为什么使用相对偏移量？这样可以节约存储空间。每条消息的绝对偏移量占用8个字节，而相对偏移量只占用4个字节（relativeOffset=offset-baseOffset）。在日志段文件滚动的条件中，有一个是：追加消息的最大偏移量和当前日志段的baseOffset的差值大于Int.MaxValue，因为如果大于这个值，4个字节就无法存储相对偏移量了。

### 偏移量索引文件的查找原理

假设要查找偏移量为230的消息，查找过程如下：

![](C:\Document\Github\interview\MQ\Kafka\img\偏移量索引文件的查找原理.png)	

首先找到baseOffset=217的日志段文件（这里使用了跳跃表的结构来加速查找）。

计算相对偏移量relativeOffset=230-217=13。

在索引文件中查找不大于13的最大相对偏移量对应的索引项，即[12,456]。

根据12对应的物理地址456，在日志文件.log中定位到准确位置。

从日志文件物理位置456继续向后查找找到相对偏移量为13，即绝对偏移量为230，物理地址为468的消息。

注意：

消息在log文件中是以批次存储的，而不是单条消息进行存储。索引文件中的偏移量保存的是该批次消息的最大偏移量，而不是最小的。

Kafka强制要求索引文件大小必须是索引项大小（8B）的整数倍，假设broker端参数log.index.size.max.bytes设置的是67，那么Kafka内部也会将其转为64，即不大于67的8的最大整数倍。

## 时间戳索引

对于时间戳索引文件，保存的是 **<时间戳，相对偏移量>** 的对应关系，文件中的时间戳和相对偏移量都是单调递增的。

查询**指定时间戳**对应的消息时， 需要配合偏移量索引文件进行查找。首先通过改进的二分查找在时间戳索引文件中找到不大于目标时间戳的索引项，然后根据索引项的相对偏移量在偏移量索引文件中查找，查找方式就是上面指定偏移量的方式。 

时间戳索引文件的索引项结构如下图所示，每个索引项记录了时间戳timestamp和相对偏移量relativeOffset的对应关系，共占用12个字节。

![](C:\Document\Github\interview\MQ\Kafka\img\时间戳索引项.png)	

### 时间戳索引文件的查找原理

假设要查找时间戳为1540的消息，查找过程如下（这里时间戳只是一个示意值）：

![](C:\Document\Github\interview\MQ\Kafka\img\时间戳索引文件的查找原理.png)	

将要查找的时间戳1540和每个日志段的最大时间戳逐一对比，直到找到最大时间戳不小于1540的日志段。（日志段的最大时间戳：获取时间戳索引文件最后一个索引项的时间戳，如果大于0，取该值；否则取日志段的最近修改时间）

找到对应的日志段后，在时间戳索引文件中使用二分查找找到不大于目标时间戳1540的最大索引项，即图中的[1530,12]，获取对应的相对偏移量12

在该日志段的偏移量索引文件中找到相对偏移量不大于12的索引项，即图中的[12，456]

在日志文件中从物理位置456开始查找时间戳不小于1540的消息

**注意：**

Kafka强制要求索引文件大小必须是索引项大小（12B）的整数倍，假设broker端参数log.index.size.max.bytes设置的是67，那么Kafka内部也会将其转为60，即不大于67的12的最大整数倍。

虽然写数据时偏移量索引文件和时间戳索引文件会同时写入一个索引项，但是两个索引项的相对偏移量不一定是一样的，这是因为：生产者生产消息时可以指定时间戳，导致一个批次中的消息，偏移量最大的对应的时间戳不一定最大，而时间戳索引文件中保存的是一个批次中最大的时间戳及对应消息的相对偏移量。

这里查找目标时间戳对应的日志段时，就无法采用跳表来快速查找了，好在日志段的最大时间戳是递增的，依次查看就行了。至于为什么不单独写一个数据结构保存最大时间戳和日志段对象的对应关系，大概是通过时间戳查找消息的操作用的很少吧。

# 日志清理

Kafka提供了两种日志清理策略：

- 日志删除（Log Retention）：按照一定的保留策略**直接删除**不符合条件的日志分段。
- 日志压缩（Log Compaction）：针对消息的key进行整合，对于有相同key的但有不同value值，只保留最后一个版本。

我们可以通过 broker端参数log.cleanup.policy来设置日志清理策略,此参数的默认值为“delete”,即采用日志删除的清理策略。如果要采用日志压缩的淸理策略,就需要将log.cleanup.policy设置为“compact”,并且还需要将log.cleaner.enable(默认值为true)设定为true。通过将log.cleanup.policy参数设置为“delete, compact”,还可以同时支持日志删除和日志压缩两种策略。日志清理的粒度可以控制到主题级别,比如与log.cleanup.policy对应的主题级别的参数为cleanup.policy。

## 日志删除

在 Kafka 的日志管理器中会有一个专门的日志删除任务来周期性地检测和删除不符合保留条件的日志分段文件，这个周期可以通过 broker 端参数 `log.retention.check.interval.ms` 来配置，默认值为300000，即5分钟。当前日志分段的保留策略有3种：

- 基于时间的保留策略
- 基于日志大小的保留策略
- 基于日志起始偏移量的保留策略。

### 基于时间

日志删除任务会检查当前日志文件中是否有保留时间超过设定的阈值来寻找可删除的日志分段文件集合，如下图所示。阈值可以通过 broker 端参数 `log.retention.hours`、`log.retention.minutes` 和 `log.retention.ms` 来配置，其中 `log.retention.ms` 的优先级最高，`log.retention.minutes` 次之，`log.retention.hours` 最低。默认情况下只配置了 `log.retention.hours` 参数，其值为168，故默认情况下日志分段文件的保留时间为7天。

![](C:\Document\Github\interview\MQ\Kafka\img\日志删除-基于时间.png)	

查找过期的日志分段文件，并不是简单地根据日志分段的最近修改时间 lastModifiedTime 来计算的，而是根**据日志分段中最大的时间戳 largestTimeStamp 来计算**的。因为日志分段的 lastModifiedTime 可以被有意或无意地修改，比如执行了 touch 操作，或者分区副本进行了重新分配，lastModifiedTime 并不能真实地反映出日志分段在磁盘的保留时间。

要获取日志分段中的最大时间戳 largestTimeStamp 的值，首先要查询该日志分段所对应的时间戳索引文件，查找时间戳索引文件中最后一条索引项，若最后一条索引项的时间戳字段值大于0，则取其值，否则才设置为最近修改时间 lastModifiedTime。

### 基于日志起始偏移量

一般情况下，日志文件的起始偏移量 logStartOffset 等于第一个日志分段的 baseOffset，但这并不是绝对的，logStartOffset 的值可以通过 DeleteRecordsRequest 请求（比如使用 KafkaAdminClient 的 deleteRecords() 方法、使用 kafka-delete-records.sh 脚本）、日志的清理和截断等操作进行修改。

![](C:\Document\Github\interview\MQ\Kafka\img\日志删除-基于日志起始偏移量.png)	

基于日志起始偏移量的保留策略的判断依据是某日志分段的下一个日志分段的起始偏移量 baseOffset 是否小于等于 logStartOffset，若是，则可以删除此日志分段。如上图所示，假设 logStartOffset 等于25，日志分段1的起始偏移量为0，日志分段2的起始偏移量为11，日志分段3的起始偏移量为23，通过如下动作收集可删除的日志分段的文件集合 deletableSegments：

- 从头开始遍历每个日志分段，日志分段1的下一个日志分段的起始偏移量为11，小于 logStartOffset 的大小，将日志分段1加入 deletableSegments。
- 日志分段2的下一个日志偏移量的起始偏移量为23，也小于 logStartOffset 的大小，将日志分段2加入 deletableSegments。
- 日志分段3的下一个日志偏移量在 logStartOffset 的右侧，故从日志分段3开始的所有日志分段都不会加入 deletableSegments。

收集完可删除的日志分段的文件集合之后的删除操作同基于日志大小的保留策略和基于时间的保留策略相同。

### 基于日志大小

日志删除任务会检查当前日志的大小是否超过设定的阈值来寻找可删除的日志分段的文件集合，如下图所示。阈值可以通过 broker 端参数 `log.retention.bytes` 来配置，默认值为-1，表示无穷大。注意 `log.retention.bytes` 配置的是 Log 中所有日志文件的总大小，而不是单个日志分段（确切地说应该为 .log 日志文件）的大小。单个日志分段的大小由 broker 端参数 `log.segment.bytes `来限制，默认值为1073741824，即 1GB。

![](C:\Document\Github\interview\MQ\Kafka\img\日志删除-基于日志大小.png)		

基于日志大小的保留策略与基于时间的保留策略类似，首先计算日志文件的总大小 size 和阈值的差值 diff，即计算需要删除的日志总大小，然后从日志文件中的第一个日志分段开始进行查找可删除的日志分段的文件集合。查找出它之后就执行删除操作，这个删除操作和基于时间的保留策略的删除操作相同。

# 日志压缩

