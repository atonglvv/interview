Kafka]中采用了多副本的机制， 这是大多数分布式系统中惯用的手法， 以此来实现水平扩展、提供容灾能力、提升可用性和可靠性等。

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

![](C:\Document\Github\interview\MQ\Kafka\img\失效副本01.png)	

可以看到主题topic-partitions中的三个分区都为under-replicated分区， 因为它们都有副本处于下线状态，即处于功能失效状态。

前面提及失效副本不仅是指处于功能失效状态的副本，处于同步失效状态的副本也可以看作失效副。

怎么判定一个分区是否有副本处于同步失效的状态呢?  Kafka从0.9.x版本开始就通过唯一的broker端参数replica.lag.time.max.ms来抉择，当ISR集合中的一个follower副本滞后leader副本的时间超过此参数指定的值时则判定为同步失败，需要将此follower副本剔除出ISR集合，具体可以参考下图。replica.lag.time.max.ms参数的默认值为10000。

![](C:\Document\Github\interview\MQ\Kafka\img\失效副本02.png)	

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





























