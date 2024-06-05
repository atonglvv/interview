Kafka中采用了多副本的机制， 这是大多数分布式系统中惯用的手法， 以此来实现水平扩展、提供容灾能力、提升可用性和可靠性等。

# 副本剖析

Kafka从0.8版本开始为分区引入了多副本机制， 通过增加副本数量来提升数据容灾能力。同时， Kafka通过多副本机制实现故障自动转移， 在Kafka集群中某个broker节点失效的情况下仍然保证服务可用。

- 副本是相对于分区而言的，即副本是特定分区的副本。
- 一个分区中包含一个或多个副本， 其中一个为leader副本，其余为follower副本， 各个副本位于不同的broker节点中。只有leader副本对外提供服务， follower副本只负责数据同步。
- 分区中的所有副本统称为AR， 而ISR是指与leader副本保持同步状态的副本集合，当然leader副本本身也是这个集合中的一员。
- LEO标识每个分区中最后一条消息的下一个位置， 分区的每个副本都有自己的LEO，ISR中最小的LEO即为HW， 俗称高水位，消费者只能拉取到HW之前的消息。

从生产者发出的一条消息首先会被写入分区的leader副本， 不过还需要等待ISR集合中的所有follower副本都同步完之后才能被认为已经提交， 之后才会更新分区的HW， 进而消费者可以消费到这条消息。

## 失效副本

在ISR集合之外，也就是处于同步失效或功能失效(比如副本处于非存活状态)的副本统称为失效副本，失效副本对应的分区也就称为同步失效分区，即under-replicated分区。

正常情况下， 我们通过kafka-topics.sh脚本的under-replicated-partitions参数来显示**主题中包含失效副本的分区**时结果会返回空。比如我们来查看一下主题topic-partitions的相关信息：

```shell
bin/kafka-topics.sh --zookeeper localhost:2181/kafka --describe --top topic-partitions --under-replicated-partitions
```

上面的示例中返回为空，紧接着我们将集群中的 brokerld为2的节点关闭，再来执行同样的命令， 结果显示如下：

![](img\失效副本01.png)	

可以看到主题topic-partitions中的三个分区都为under-replicated分区， 因为它们都有副本处于下线状态，即处于功能失效状态。

前面提及失效副本不仅是指处于功能失效状态的副本，处于同步失效状态的副本也可以看作失效副本。

怎么判定一个分区是否有副本处于同步失效的状态呢?  Kafka从0.9.x版本开始就通过唯一的broker端参数replica.lag.time.max.ms来抉择，当ISR集合中的一个follower副本滞后leader副本的时间超过此参数指定的值时则判定为同步失败，需要将此follower副本剔除出ISR集合，具体可以参考下图。replica.lag.time.max.ms参数的默认值为10000。

![](img\失效副本02.png)

具体的实现原理也很容易理解，**当follower副本将leader副本LEO(Log End Offset) 之前的日志全部同步时**， 则认为该follower副本已经追赶上leader副本， 此时**更新该副本的lastCaughtUpTimeMs标识**。

Kafka的副本管理器会启动一个副本过期检测的定时任务， 而这个定时任务会定时检查当前时间与副本的lastCaughtUpTimeMs差值是否大于参数replica.lag.time.max.ms指定的值。

千万不要错误地认为follower副本只要拉取leader副本的数据就会更新lastCaughtUpTimeMs。试想一下，当leader副本中消息的流入速度大于follower副本中拉取的速度时， 就算follower副本一直不断地拉取leader副本的消息也不能与leader副本同步。如果还将此follower副本置于ISR集合中， 那么当leader副本下线而选取此follower副本为新的leader副本时就会造成消息的严重丢失。

Kafka源码注释中说明了一般有两种情况会导致副本失效：

- follower副本进程卡住， 在一段时间内根本没有向leader副本发起同步请求， 比如频繁的Full GC。
- follower副本进程同步过慢，在一段时间内都无法追赶上leader副本， 比如I/O开销过大。

在这里再补充一点，如果通过工具增加了副本因子，那么新增加的副本在赶上leader副本之前也都是处于失效状态的。如果一个follower副本由于某些原因(比如宕机)而下线，之后又上线， 在追赶上leader副本之前也处于失效状态。

在0.9.x版本之前， Kafka中还有另一个参数replica.lag.max.messages(默认值为4000) ，它也是用来判定失效副本的，当一个follower副本滞后leader副本的消息数超过replica.lag.max.messages的大小时，则判定它处于同步失效的状态。它与replica.lag.time.max.ms参数判定出的失效副本取并集组成一个失效副本的集合， 从而进一步剥离出分区的ISR集合。不过这个replica.lag.max.messages参数很难给定一个合适的值， 若设置得太大， 则这个参数本身就没有太多意义， 若设置得太小则会让follower副本反复处于同步、未同步、同步的死循环中， 进而又造成ISR集合的频繁伸缩。而且这个参数是broker级别的， 也就是说， 对broker中的所有主题都生效。以默认的值4000为例， 对于消息流入速度很低的主题(比如TPS为10) ，这个参数并无用武之地； 而对于消息流入速度很高的主题(比如TPS为20000) ， 这个参数的取值又会引入ISR的频繁变动。所以从0.9.x版本开始， Kafka就彻底移除了这一参数。

具有失效副本的分区可以从侧面反映出Kafka集群的很多问题， 毫不夸张地说：如果只用一个指标来衡量Kafka， 那么同步失效分区(具有失效副本的分区) 的个数必然是首选。

## ISR的伸缩

Kafka在启动的时候会开启两个与ISR相关的定时任务， 名称分别为**isr-expiration**和**isr-change-propagation**。**isr-expiration任务会周期性地检测每个分区是否需要缩减其ISR集合。这个周期和replica.lag.time.max.ms参数有关，大小是这个参数值的一半， 默认值为5000ms。当检测到ISR集合中有失效副本时，就会收缩ISR集合。**如果某个分区的ISR集合发生变更， 则会将变更后的数据记录到ZooKeeper对应的**/brokers/topics/<topic>/partition/<partition>/state**节点中。节点中的数据示例如下：

```json
{
    "controller_epoch": 26,
    "leader": 0,
    "version": 1,
    "leader_epoch": 2,
    "isr": [
        0,
        1
    ]
}
```

其中controller_epoch表示当前Kafka控制器的epoch， leader表示当前分区的leader副本所在的broker的id编号， version表示版本号(当前版本固定为1) ，leader_epoch表示当前分区的leader纪元， isr表示变更后的ISR列表。

除此之外， 当ISR集合发生变更时还会将变更后的记录缓存到isrChangeSet中，isr-change-propagation任务会周期性(固定值为2500ms) 地检查isrChangeSet， 如果发现isrChangeSet中有ISR集合的变更记录， 那么它会在ZooKeeper的/isr_change_notification路径下创建一个以isr_change_开头的持久顺序节点(比如/isr_change_notification/isr_change_0000000000) ， 并将isChangeSet中的信息保存到这个节点中。Kafka控制器为/isr_change_notification添加了一个Watcher，当这个节点中有子节点发生变化时会触发Watcher的动作， 以此通知控制器更新相关元数据信息并向它管理的broker节点发送更新元数据的请求， 最后删除/isr_change_notification路径下已经处理过的节点。频繁地触发Watcher会影响Kafka控制器、ZooKeeper甚至其他broker节点的性能。为了避免这种情况， Kafka添加了限定条件， 当检测到分区的ISR集合发生变化时， 还需要检查以下两个条件：

- 上一次ISR集合发生变化距离现在已经超过5s。
- 上一次写入ZooKeeper的时间距离现在已经超过60s。

满足以上两个条件之一才可以将ISR集合的变化写入目标节点。

有缩减对应就会有扩充， 那么Kafka又是何时扩充ISR的呢?

随着follower副本不断与leader副本进行消息同步， follower副本的LEO也会逐渐后移，并最终追赶上leader副本， 此时该follower副本就有资格进入ISR集合。追赶上leader副本的判定准则是此副本的LEO是否不小于leader副本的HW， 注意这里并不是和leader副本的LEO相比。ISR扩充之后同样会更新ZooKeeper中的**/brokers/topics/<topic>/partition/<partition>/state**节点和isrChangeSet，之后的步骤就和ISR收缩时的相同。

当ISR集合发生增减时， 或者ISR集合中任一副本的LEO发生变化时，都可能会影响整个分区的HW。

如下图所示， leader副本的LEO为9， follower1副本的LEO为7 而follower2副本的LEO为6， 如果判定这3个副本都处于ISR集合中， 那么这个分区的HW为6；如果follower3 已经被判定为失效副本被剥离出ISR集合， 那么此时分区的HW为leader副本和follower1副本中LEO的最小值， 即为7。

![](img\ISR的伸缩01.png)

冷门知识：很多读者对Kafka中的HW的概念并不陌生， 但是却并不知道还有一个LW的概念。LW是Low Watermark的缩写，俗称“低水位”，代表AR集合中最小的logStartOffset值。副本的拉取请求(FetchRequest，它有可能触发新建日志分段而旧的被清理， 进而导致logStartOffset的增加)和删除消息请求(DeleteRecordRequest) 都有可能促使LW的增长。

## LEO与HW

对于副本而言，还有两个概念：本地副本(Local Replica) 和远程副本(Remote Replica) ，本地副本是指对应的Log分配在当前的broker节点上，远程副本是指对应的Log分配在其他的broker节点上。在Kafka中，同一个分区的信息会存在多个broker节点上，并被其上的副本管理器所管理，这样在逻辑层面每个broker节点上的分区就有了多个副本，但是只有本地副本才有对应的日志。如下图所示，某个分区有3个副本分别位于broker0、broker1和broker2节点中，其中带阴影的方框表示本地副本。假设broker0上的副本1为当前分区的leader副本， 那么副本2和副本3就是follower副本， 整个消息追加的过程可以概括如下：

1. 生产者客户端发送消息至leader副本(副本1) 中。
2. 消息被追加到leader副本的本地日志，并且会更新日志的偏移量。
3. follower副本(副本2和副本3) 向leader副本请求同步数据。
4. leader副本所在的服务器读取本地日志，并更新对应拉取的follower副本的信息。
5. leader副本所在的服务器将拉取结果返回给follower副本。
6. follower副本收到leader副本返回的拉取结果， 将消息追加到本地日志中， 并更新日志的偏移量信息。

![](img\LEO与HW01.png)

了解了这些内容后，我们再来分析在这个过程中各个副本LEO和HW的变化情况。下面的示例采用同图8-3中相同的环境背景， 如图8-4所示， 生产者一直在往leader副本(带阴影的方框) 中写入消息。某一时刻， leader副本的LEO增加至5， 并且所有副本的HW还都为0。之后follower副本(不带阴影的方框) 向leader副本拉取消息， 在拉取的请求中会带有自身的LEO信息， 这个LEO信息对应的是FetchRequest请求中的fetch_offset。leader副本返回给follower副本相应的消息， 并且还带有自身的HW信息， 如图8-5所示， 这个HW信息对应的是FetchResponse中的high_watermark。

![](img\LEO与HW02.png)	

接下来follower副本再次请求拉取leader副本中的消息， 如图8-6所示。

此时leader副本收到来自follower副本的FetchRequest请求， 其中带有LEO的相关信息，选取其中的最小值作为新的HW，即min(15,3,4) =3。然后连同消息和HW一起返回FetchResponse给follower副本， 如图8-7所示。注意leader副本的HW是一个很重要的东西， 因为它直接影响了分区数据对消费者的可见性。

![](img\LEO与HW03.png)	

两个follower副本在收到新的消息之后更新LEO并且更新自己的HW为3(min(LEO,3) =3) 。

在一个分区中， leader副本所在的节点会记录所有副本的LEO， 而follower副本所在的节点只会记录自身的LEO， 而不会记录其他副本的LEO。对HW而言， 各个副本所在的节点都只记录它自身的HW。变更图8-3， 使其带有相应的LEO和HW信息， 如图8-8所示。leader副本中带有其他follower副本的LEO， 那么它们是什么时候更新的呢?leader副本收到follower副本的FetchRequest请求之后， 它首先会从自己的日志文件中读取数据， 然后在返回给follower副本数据前先更新follower副本的LEO。

![](img\LEO与HW04.png)

Kafka的根目录下有cleaner-offset-checkpoint、log-start-offset-checkpoint、recovery-point-offset-checkpoint和replication-offset-checkpoint四个检查点文件。

recovery-point-offset-checkpoint和replication-offset-checkpoint这两个文件分别对应了LEO和HW。Kafka中会有一个定时任务负责将所有分区的LEO刷写到恢复点文件recovery-point-offset-checkpoint中， 定时周期由broker端参数log.flush.offset.checkpoint.interval.ms来配置， 默认值为60000。还有一个定时任务负责将所有分区的HW刷写到复制点文件replication-offset-checkpoint中， 定时周期由broker端参数replica.high.watermark.checkpoint.interval.ms来配置， 默认值为5000。

log-start-offset-checkpoint文件对应logStartOffset(注意不能缩写为LSO， 因为在Kafka中LSO是LastStableOffset的缩写) ，这个在5.4.1节中就讲过， 在FetchRequest和FetchResponse中也有它的身影， 它用来标识日志的起始偏移量。各个副本在变动LEO和HW的过程中，logStartOffset也有可能随之而动。Kafka也有一个定时任务来负责将所有分区的logStartOffset书写到起始点文件log-start-offset-checkpoint中， 定时周期由broker端参数log.flush.start.offset.checkpoint.interval.ms来配置， 默认值为60000。

## Leader Epoch的介入

 如果leader副本发生切换， 那么同步过程又该如何处理呢?在0.11.0.0版本之前， Kafka使用的是基于HW的同步机制， 但这样有可能出现数据丢失或leader副本和follower副本数据不一致的问题。

首先我们来看一下数据丢失的问题， 如图8-9所示， ReplicaＢ是当前的leader副本(用L标记) ， Replica A是follower副本。参照8.1.3节中的图8-4至图8-7的过程来进行分析：在某一时刻， B中有2条消息m1和m 2， A从B中同步了这两条消息， 此时A和B的LEO都为2，同时HW都为1； 之后A再向B中发送请求以拉取消息， FetchRequest请求中带上了A的LEO信息，B在收到请求之后更新了自己的HW为2；B中虽然没有更多的消息，但还是在延时一段时间之后(参考6.3节中的延时拉取) 返回FetchResponse， 并在其中包含了HW信息； 最后A根据FetchResponse中的HW信息更新自己的HW为2。

![](img\LeaderEpoch的介入01.png)

可以看到整个过程中两者之间的HW同步有一个间隙， 在Ａ写入消息m2之后(LEO更新为2) 需要再一轮的FetchRequest/FetchResponse才能更新自身的HW为2。如图8-10所示， 如果在这个时候A宕机了，那么在A重启之后会根据之前HW位置(这个值会存入本地的复制点文件replication-offset-checkpoint) 进行日志截断， 这样便会将m2这条消息删除， 此时Ａ只剩下m1这一条消息， 之后A再向B发送FetchRequest请求拉取消息。

![](img\LeaderEpoch的介入02.png)

此时若B再宕机， 那么A就会被选举为新的leader， 如图8-11所示。B恢复之后会成为follower， 由于follower副本HW不能比leader副本的HW高， 所以还会做一次日志截断， 以此将HW调整为1。这样一来m2这条消息就丢失了(就算B不能恢复，这条消息也同样丢失)。

![](img\LeaderEpoch的介入03.png)

对于这种情况，也有一些解决方法，比如等待所有follower副本都更新完自身的HW之后再更新leader副本的HW， 这样会增加多一轮的FetchRequest/FetchResponse延迟， 自然不够妥当。还有一种方法就是follower副本恢复之后， 在收到leader副本的FetchResponse前不要截断follower副本(follower副本恢复之后会做两件事情：截断自身和向leader发送Fetch Request请求)，不过这样也避免不了数据不一致的问题。

如图8-12所示，当前leader副本为A， follower副本为B， A中有2条消息m1和m2， 并且HW和LEO都为2， B中有1条消息m1， 并且HW和LEO都为1.假设A和B同时“挂掉”，然后B第一个恢复过来并成为leader， 如图8-13所示。

![](img\LeaderEpoch的介入04.png)

之后Ｂ写入消息m3， 并将LEO和HW更新至2(假设所有场景中的min.insync.replicas参数配置为1) 。此时Ａ也恢复过来了， 根据前面数据丢失场景中的介绍可知它会被赋予follower的角色， 并且需要根据HW截断日志及发送FetchRequest至B， 不过此时A的HW正好也为2，那么就可以不做任何调整了，如图8-14所示。

![](img\LeaderEpoch的介入05.png)

如此一来A中保留了m2而Ｂ中没有，Ｂ中新增了m3而Ａ也同步不到，这样A和B就出现了数据不一致的情形。

为了解决上述两种问题， Kafka从0.11.0.0开始引入了**leader epoch**的概念， 在需要截断数据的时候使用leader epoch作为参考依据而不是原本的HW。leader epoch代表leader的纪元信息(epoch) ， 初始值为0。每当leader变更一次， leader epoch的值就会加1， 相当于为leader增设了一个版本号。与此同时， 每个副本中还会增设一个矢量<LeaderEpoch=>StartOffset>， 其中StartOffset表示当前LeaderEpoch下写入的第一条消息的偏移量。每个副本的Log下都有一个leader-epoch-checkpoint文件， 在发生leader epoch变更时， 会将对应的矢量对追加到这个文件中，其实这个文件在图5-2中已有所呈现。5.2.5节中讲述v2版本的消息格式时就提到了消息集中的partition leader epoch字段， 而这个字段正对应这里讲述的leader epoch。

下面我们再来看一下引入leader epoch之后如何应付前面所说的数据丢失和数据不一致的场景。首先讲述应对数据丢失的问题， 如图8-15所示， 这里只比图8-9中多了LE(Leader Epoch的缩写，当前Ａ和Ｂ中的LE都为0)。

![](img\LeaderEpoch的介入06.png)

同样A发生重启， 之后Ａ不是先忙着截断日志而是先发送OffsetsForLeaderEpochRequest请求给B(OffsetsForLeaderEpochRequest请求体结构如图8-16所示， 其中包含A当前的Leader Epoch值) ， B作为目前的leader在收到请求之后会返回当前的LEO(Log End Offset， 注意图中LE0和LEO的不同) ， 与请求对应的响应为OffsetsForLeaderEpochResponse， 对应的响应体结构可以参考图8-17，整个过程可以参考图8-18。

![](img\LeaderEpoch的介入07.png)

如果A中的LeaderEpoch(假设为LE_A) 和Ｂ中的不相同， 那么Ｂ此时会查找LeaderEpoch为LEA+1对应的StartOffset并返回给A， 也就是LEA对应的LEO， 所以我们可以将OffsetsForLeaderEpochRequest的请求看作用来查找follower副本当前LeaderEpoch的LEO。

![](img\LeaderEpoch的介入08.png)

如图8-18所示， A在收到2之后发现和目前的LEO相同， 也就不需要截断日志了。之后同图8-11所示的一样， Ｂ发生了宕机， A成为新的leader， 那么对应的LE=0也变成了LE=1，对应的消息m2此时就得到了保留，这是原本图8-11中所不能的，如图8-19所示。之后不管B有没有恢复， 后续的消息都可以以LE1为LeaderEpoch陆续追加到A中。

![](img\LeaderEpoch的介入09.png)

下面我们再来看一下leader epoch如何应对数据不一致的场景。如图8-20所示， 当前A为leader， B为follower， A中有2条消息m1和m 2， 而Ｂ中有1条消息m1。假设A和B同时“挂掉”， 然后Ｂ第一个恢复过来并成为新的leader。

![](img\LeaderEpoch的介入10.png)

之后Ｂ写入消息m3， 并将LEO和HW更新至2， 如图8-21所示。注意此时的LeaderEpoch已经从LE0增至LE1了。

![](img\LeaderEpoch的介入11.png)

紧接着Ａ也恢复过来成为follower并向B发送OffsetsForLeaderEpochRequest请求， 此时Ａ的LeaderEpoch为LE0。B根据LE0查询到对应的offset为1并返回给A， A就截断日志并删除了消息m2， 如图8-22所示。之后A发送FetchRequest至B请求来同步数据， 最终A和B中都有两条消息m1和m3， HW和LEO都为2， 并且Leader Epoch都为LE1， 如此便解决了数据不一致的问题。

![](img\LeaderEpoch的介入12.png)

## 为什么不支持读写分离

在Kafka中， 生产者写入消息、消费者读取消息的操作都是与leader副本进行交互的， 从而实现的是一种主写主读的生产消费模型。数据库、Redis等都具备主写主读的功能， 与此同时还支持主写从读的功能，主写从读也就是读写分离，为了与主写主读对应，这里就以主写从读来称呼。Kafka并不支持主写从读， 这是为什么呢?

从代码层面上来说，虽然增加了代码复杂度，但在Kafka中这种功能完全可以支持。对于这个问题，我们可以从“收益点”这个角度来做具体分析。主写从读可以让从节点去分担主节点的负载压力，预防主节点负载过重而从节点却空闲的情况发生。但是主写从读也有2个很明显的缺点：

(1) 数据一致性问题。
(2) 延时问题。类似Redis这种组件， 数据从写入主节点到同步至从节点中的过程需要经历网络→主节点内存→网络→从节点内存这几个阶段， 整个过程会耗费一定的时间。而在Kafka中， 主从同步会比Redis更加耗时， 它需要经历网络→主节点内存→主节点磁盘→网络→从节点内存→从节点磁盘这几个阶段。对延时敏感的应用而言，主写从读的功能并不太适用。

现实情况下，很多应用既可以忍受一定程度上的延时，也可以忍受一段时间内的数据不一致的情况， 那么对于这种情况， Kafka是否有必要支持主写从读的功能呢？

主读从写可以均摊一定的负载却不能做到完全的负载均衡，比如对于数据写压力很大而读压力很小的情况， 从节点只能分摊很少的负载压力， 而绝大多数压力还是在主节点上。而在Kafka中却可以达到很大程度上的负载均衡，而且这种均衡是在主写主读的架构上实现的。我们来看一下Kafka的生产消费模型， 如图8-23所示。

![](img\为什么不支持读写分离01.png)

如图8-23 所示，在Kafka 集群中有3个分区，每个分区有3个副本，正好均匀地分布在3个broker上， 灰色阴影的代表leader副本， 非灰色阴影的代表follower副本， 虚线表示follower副本从leader副本上拉取消息。当生产者写入消息的时候都写入leader副本， 对于图8-23中的情形， 每个broker都有消息从生产者流入； 当消费者读取消息的时候也是从leader副本中读取的， 对于图8-23中的情形， 每个broker都有消息流出到消费者。

我们很明显地可以看出， 每个broker上的读写负载都是一样的， 这就说明Kafka可以通过主写主读实现主写从读实现不了的负载均衡。图8-23展示是一种理想的部署情况，有以下几种情况(包含但不仅限于)会造成一定程度上的负载不均衡：
(1) broker端的分区分配不均。当创建主题的时候可能会出现某些broker分配到的分区数多而其他broker分配到的分区数少， 那么自然而然地分配到的leader副本也就不均。
(2) 生产者写入消息不均。生产者可能只对某些broker中的leader副本进行大量的写入操作， 而对其他broker中的leader副本不闻不问。
(3) 消费者消费消息不均。消费者可能只对某些broker中的leader副本进行大量的拉取操作， 而对其他broker中的leader副本不闻不问。
(4) leader副本的切换不均。在实际应用中可能会由于broker宕机而造成主从副本的切换，或者分区副本的重分配等， 这些动作都有可能造成各个broker中leader副本的分配不均。

对此，我们可以做一些防范措施。针对第一种情况，在主题创建的时候尽可能使分区分配得均衡， 好在Kafka中相应的分配算法也是在极力地追求这一目标， 如果是开发人员自定义的分配，则需要注意这方面的内容。对于第二和第三种情况，主写从读也无法解决。对于第四种情况， Kafka提供了优先副本的选举来达到leader副本的均衡， 与此同时， 也可以配合相应的监控、告警和运维平台来实现均衡的优化。

在实际应用中， 配合监控、告警、运维相结合的生态平台， 在绝大多数情况下Kafka都能做到很大程度上的负载均衡。总的来说， Kafka只支持主写主读有几个优点：可以简化代码的实现逻辑，减少出错的可能；将负载粒度细化均摊，与主写从读相比，不仅负载效能更好，而且对用户可控；没有延时的影响；在副本稳定的情况下，不会出现数据不一致的情况。为此，Kafka又何必再去实现对它而言毫无收益的主写从读的功能呢?这一切都得益于Kafka优秀的架构设计，从某种意义上来说，主写从读是由于设计上的缺陷而形成的权宜之计。

# 日志同步机制

在分布式系统中，日志同步机制既要保证数据的一致性，也要保证数据的顺序性。虽然有许多方式可以实现这些功能， 但最简单高效的方式还是从集群中选出一个leader来负责处理数据写入的顺序性。只要leader还处于存活状态， 那么follower只需按照leader中的写入顺序来进行同步即可。

通常情况下， 只要leader不宕机我们就不需要关心follower的同步问题。不过当leader宕机时， 我们就要从follower中选举出一个新的leader。follower的同步状态可能落后leader很多，甚至还可能处于宕机状态， 所以必须确保选择具有最新日志消息的follower作为新的leader。日志同步机制的一个基本原则就是：如果告知客户端已经成功提交了某条消息， 那么即使leader宕机， 也要保证新选举出来的leader中能够包含这条消息。这里就有一个需要权衡(tradeoff)的地方， 如果leader在消息被提交前需要等待更多的follower确认， 那么在它宕机之后就可以有更多的follower替代它， 不过这也会造成性能的下降。

对于这种tradeoff， 一种常见的做法是“少数服从多数”， 它可以用来负责提交决策和选举决策。虽然Kafka不采用这种方式， 但可以拿来探讨和理解tradeoff的艺术。在这种方式下， 如果我们有2f+1个副本，那么在提交之前必须保证有f+1个副本同步完消息。同时为了保证能正确选举出新的leader， 至少要保证有f+1个副本节点完成日志同步并从同步完成的副本中选举出新的leader节点。并且在不超过f个副本节点失败的情况下， 新的leader需要保证不会丢失已经提交过的全部消息。这样在任意组合的f+1个副本中，理论上可以确保至少有一个副本能够包含已提交的全部消息， 这个副本的日志拥有最全的消息， 因此会有资格被选举为新的leader来对外提供服务。

“少数服从多数”的方式有一个很大的优势，系统的延迟取决于最快的几个节点，比如副本数为3， 那么延迟就取决于最快的那个follower而不是最慢的那个(除了leader， 只需要另一个follower确认即可) 。不过它也有一些劣势， 为了保证leader选举的正常进行， 它所能容忍的失败follower数比较少， 如果要容忍1个follower失败， 那么至少要有3个副本， 如果要容忍2个follower失败， 必须要有5个副本。也就是说， 在生产环境下为了保证较高的容错率，必须要有大量的副本，而大量的副本又会在大数据量下导致性能的急剧下降。这也就是“少数服从多数”的这种Quorum模型常被用作共享集群配置(比如ZooKeeper) ， 而很少用于主流的数据存储中的原因。

与“少数服从多数”相关的一致性协议有很多， 比如Zab、Raft和Viewstamped Replication等。而Kafka使用的更像是微软的PacificA算法。

在Kafka中动态维护着一个ISR集合， 处于ISR集合内的节点保持与leader相同的高水位(HW) ， 只有位列其中的副本(unclean.leader.election.enable配置为false) 才有资格被选为新的leader。写入消息时只有等到所有ISR集合中的副本都确认收到之后才能被认为已经提交。位于ISR中的任何副本节点都有资格成为leader， 选举过程简单(详细内容可以参考6.4.3节) 、开销低， 这也是Kafka选用此模型的重要因素。Kafka中包含大量的分区， leader副本的均衡保障了整体负载的均衡， 所以这一因素也极大地影响Kafka的性能指标。

在采用ISR模型和(f+1) 个副本数的配置下，一个Kafka分区能够容忍最大f个节点失败，相比于“少数服从多数”的方式所需的节点数大幅减少。实际上，为了能够容忍f个节点失败，

“少数服从多数”的方式和ISR的方式都需要相同数量副本的确认信息才能提交消息。比如，为了容忍1个节点失败， “少数服从多数”需要3个副本和1个follower的确认信息， 采用ISR的方式需要2个副本和1个follower的确认信息。在需要相同确认信息数的情况下， 采用ISR的方式所需要的副本总数变少，复制带来的集群开销也就更低，“少数服从多数”的优势在于它可以绕开最慢副本的确认信息， 降低提交的延迟， 而对Kafka而言， 这种能力可以交由客户端自己去选择。

另外，一般的同步策略依赖于稳定的存储系统来做数据恢复，也就是说，在数据恢复时日志文件不可丢失且不能有数据上的冲突。不过它们忽视了两个问题：首先，磁盘故障是会经常发生的，在持久化数据的过程中并不能完全保证数据的完整性；其次，即使不存在硬件级别的故障， 我们也不希望在每次写入数据时执行同步刷盘(fsync) 的动作来保证数据的完整性， 这样会极大地影响性能。而Kafka不需要宕机节点必须从本地数据日志中进行恢复， Kafka的同步方式允许宕机副本重新加入ISR集合， 但在进入ISR之前必须保证自己能够重新同步完leader中的所有数据。

# 可靠性分析

很多人问过笔者类似这样的 些问题：怎样可以确保 Kafka 完全可靠？如果这样做就可以确保消息不丢失了吗？笔者认为 就可靠性本身而言，它并不是一个可以用简单的 ”或“ 否”来衡量的 个指标，而一般是采用几9个来衡量的。任何东西不可能做到完全的可靠，即使能应付单机故障，也难以应付集群、数据中心等集体故障，即使躲得过天灾也未必躲得过人祸就可靠性而言，我们可以基于一定的假设前提来做分析本节要讲述的是：在只考虑 Kafka本身使用方式的前提下如何最大程度地提高可靠性。

就Kafka 而言，越多的副本数越能够保证数据的可靠性，副本数可以在创建主题时配置，也可以在后期修改，不过副本数越多也会引起磁盘、网络带宽的浪费，同时会引起性能的下降。一般而言，设置副本数为3即可满足绝大多数场景对可靠性的要求，而对可靠性要求更高的场景下 ，可以适当增大这个数值，比如国内部分银行在使用 Kafka 时就会设置副本数为5。与此同时，如果能够在分配分区副本的时候引入基架信息 broker.rack 参数） ，那么还要应对机架整体岩机的风险。

仅依靠副本数来支撑可靠性是远远不够的，大多数人还会想到生产者客户端参数 acks。在2.3 节中我们就介绍过这个参数：相比于 acks = -1 （客户端还可以配置为all ，它的含义与-1 一样，以下只以-1 来进行陈述)可以最大程度地提高消息的可靠性。

对于 acks = 1的配置，生产者将消息发送到 leader 副本， leader 副本在成功写入本地日志之后会告知生产者己经成功提交，如图 8-24 所示 如果此时 ISR 集合的 follower 副本还没来得及拉取到 leader 中新写入的消息， leader 就看机了，那么此次发迭的消息就会丢失。
![](img\可靠性分析01.png)
对于 ack = -1配置，生产者将消息发送到 leader 副本， leader 副本在成功写入本地日志之后还要等待 ISR 中的 follower 副本全部同步完成才能够告知生产者已经成功提交，即使此时leader 副本宕机，消息也不会丢失，如图 8-25 所示。

![](img\可靠性分析02.png)

同样对于 acks = -1的配 ，如果在消息成功写入 leader 副本之后，并且在被 ISR 中的所有副本同步之前 leader 副本宕机了，那么生产者会收到异常以此告知此次发送失败，如图 8-26 所示。

![](img\可靠性分析03.png)

在2.1.2节中，我们讨论了消息发送的3种模式，即发后即忘、同步和异步。对于发后即忘的模式，不管消息有没有被成功写入，生产者都不会收到通知，那么即使消息写入失败也无从得知，因此发后即忘的模式不适合高可靠性要求的场景。如果要提升可靠性，那么生产者可以采用同步或异步的模式，在出现异常情况时可以及时获得通知，以便可以做相应的补救措施，比如选择重试发送(可能会引起消息重复)。

有些发送异常属于可重试异常， 比如Network Exception， 这个可能是由瞬时的网络故障而导致的，一般通过重试就可以解决。对于这类异常，如果直接抛给客户端的使用方也未免过于兴师动众，客户端内部本身提供了重试机制来应对这种类型的异常，通过retries参数即可配置。默认情况下，retries参数设置为0， 即不进行重试， 对于高可靠性要求的场景， 需要将这个值设置为大于0的值，在2.3节中也谈到了与retries参数相关的还有一个retry.back off.ms参数， 它用来设定两次重试之间的时间间隔， 以此避免无效的频繁重试。在配置retries和retry.back off.ms之前， 最好先估算一下可能的异常恢复时间， 这样可以设定总的重试时间大于这个异常恢复时间，以此来避免生产者过早地放弃重试。如果不知道retries参数应该配置为多少， 则可以参考KafkaAdminClient， 在KafkaAdminClient中retries参数的默认值为5。

注意如果配置的retries参数值大于0， 则可能引起一些负面的影响。首先同2.3节中谈及的一样， 由于默认的max.in.flight.requests.per.connection参数值为5， 这样可能会影响消息的顺序性，对此要么放弃客户端内部的重试功能，要么将max.in.flight.requests.per.connection参数设置为1， 这样也就放弃了吞吐。其次，有些应用对于时延的要求很高， 很多时候都是需要快速失败的， 设置retries>0会增加客户端对于异常的反馈时延，如此可能会对应用造成不良的影响。

我们回头再来看一下acks=-1的情形， 它要求ISR中所有的副本都收到相关的消息之后才能够告知生产者已经成功提交。试想一下这样的情形， leader副本的消息流入速度很快， 而follower副本的同步速度很慢， 在某个临界点时所有的follower副本都被剔除出了ISR集合， 那么ISR中只有一个leader副本， 最终acks=-1演变为acks=1的情形， 如此也就加大了消息丢失的风险。Kafka也考虑到了这种情况， 并为此提供了min.insync.replicas参数(默认值为1) 来作为辅助(配合acks=-1来使用) ， 这个参数指定了IS R集合中最小的副本数， 如果不满足条件就会抛出NotEnoughReplicasException或NotEnoughReplicasAfterAppendException。在正常的配置下， 需要满足副本数 > min.insync.replicas参数的值。一个典型的配置方案为：副本数配置为3， min.insync.replicas参数值配置为2。注意min.insync.replicas参数在提升可靠性的时候会从侧面影响可用性。试想如果IS R中只有一个leader副
本， 那么最起码还可以使用， 而此时如果配置min.insync.replicas>1， 则会使消息无法写入。

与可靠性和ISR集合有关的还有一个参数——unclean.leader.election.enable。这个参数的默认值为false， 如果设置为true就意味着当leader下线时候可以从非IS R集合中选举出新的leader， 这样有可能造成数据的丢失。如果这个参数设置为false， 那么也会影响可用性， 非ISR集合中的副本虽然没能及时同步所有的消息， 但最起码还是存活的可用副本。随着Kafka版本的变更， 有的参数被淘汰， 也有新的参数加入进来， 而传承下来的参数一般都很少会修改既定的默认值，而unclean.leader.election.enable就是这样一个反例， 从0.11.0.0版本开始， unclean.leader.election.enable的默认值由原来的true改为了false， 可以看出Kafka的设计者愈发地偏向于可靠性的提升。

在broker端还有两个参数log.flush.interval.messages和log.flush.interval.ms，用来调整同步刷盘的策略，默认是不做控制而交由操作系统本身来进行处理。同步刷盘是增强一个组件可靠性的有效方式， Kafka也不例外， 但笔者对同步刷盘有一定的疑问——绝大多数情景下，一个组件(尤其是大数据量的组件)的可靠性不应该由同步刷盘这种极其损耗性能的操作来保障，而应该采用多副本的机制来保障。

对于消息的可靠性， 很多人都会忽视消费端的重要性， 如果一条消息成功地写入Kafka，并且也被Kafka完好地保存， 而在消费时由于某些疏忽造成没有消费到这条消息， 那么对于应用来说，这条消息也是丢失的。

enable.auto.commit参数的默认值为true， 即开启自动位移提交的功能， 虽然这种方式非常简便，但它会带来重复消费和消息丢失的问题，对于高可靠性要求的应用来说显然不可取，所以需要将enable.auto.commit参数设置为false来执行手动位移提交。在执行手动位移提交的时候也要遵循一个原则：如果消息没有被成功消费，那么就不能提交所对应的消费位移。对于高可靠要求的应用来说，宁愿重复消费也不应该因为消费异常而导致消息丢失。有时候，由于应用解析消息的异常，可能导致部分消息一直不能够成功被消费，那么这个时候为了不影响整体消费的进度，可以将这类消息暂存到死信队列(查看11.3节)中，以便后续的故障排除。

对于消费端， Kafka还提供了一个可以兜底的功能， 即回溯消费， 通过这个功能可以让我们能够有机会对漏掉的消息相应地进行回补，进而可以进一步提高可靠性。











