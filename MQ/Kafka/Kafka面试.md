**Kafka 分区的目的？**

分区对于 Kafka 集群的好处是：实现负载均衡。

分区对于消费者来说，可以提高并发度，提高效率。



**kafka的消费者是pull(拉)还是push(推)模式，这种模式有什么好处？**

producer 将消息推送到 broker，consumer 从broker 拉取消息。

优点：pull模式消费者自主决定是否批量从broker拉取数据，而push模式在无法知道消费者消费能力情况下，不易控制推送速度，太快可能造成消费者奔溃，太慢又可能造成浪费。

缺点：如果 broker 没有可供消费的消息，将导致 consumer 不断在循环中轮询，直到新消息到到达。为了避免这点，Kafka 有个参数可以让 consumer阻塞知道新消息到达(当然也可以阻塞知道消息的数量达到某个特定的量这样就可以批量发送)。



**Kafka在什么情况下会消息丢失？**

以下几个阶段，都有可能会出现消息丢失的情况

1）消息发送的时候，如果发送出去以后，消息可能因为网络问题并没有发送成功。

2）消息消费的时候，消费者在消费消息的时候，若还未做处理的时候，服务挂了，那这个消息不就丢失了。

3）分区中的leader所在的broker挂了之后。

我们知道，Kafka的Topic中的分区Partition是leader与follower的主从机制，发送消息与消费消息都直接面向leader分区，并不与follower交互，follower则会去leader中拉取消息，进行消息的备份，这样保证了一定的可靠性。

但是，当leader副本所在的broker突然挂掉，那么就要从follower中选举一个leader，但leader的数据在**挂掉之前并没有同步到follower的这部分消息**肯定就会丢失掉。



**谈一谈 Kafka 的再均衡**

在Kafka中，当有新消费者加入或者订阅的topic数发生变化时，会触发Rebalance(再均衡：在同一个消费者组当中，分区的所有权从一个消费者转移到另外一个消费者)机制，Rebalance顾名思义就是重新均衡消费者消费。Rebalance的过程如下：

第一步：所有成员都向coordinator发送请求，请求入组。一旦所有成员都发送了请求，coordinator会从中选择一个consumer担任leader的角色，并把组成员信息以及订阅信息发给leader。

第二步：leader开始分配消费方案，指明具体哪个consumer负责消费哪些topic的哪些partition。一旦完成分配，leader会将这个方案发给coordinator。coordinator接收到分配方案之后会把方案发给各个consumer，这样组内的所有成员就都知道自己应该消费哪些分区了。

所以对于Rebalance来说，Coordinator起着至关重要的作用。



**Kafka的应用场景**

1）**日志信息收集记录**我个人接触的项目中，Kafka使用最多的场景，就是用它与**FileBeats**和**ELK**组成典型的**日志收集、分析处理以及展示的框架**

![](\img\面试01.jpg)	

该图为**FileBeats+Kafka+ELK集群架构。**Kafka在框架中，作为消息缓冲队列

FileBeats先将数据传递给消息队列，Logstash server（二级Logstash）拉取消息队列中的数据，进行过滤和分析，然后将数据传递给Elasticsearch进行存储，最后，再由Kibana将日志和数据呈现给用户

由于引入了Kafka缓冲机制，即使远端**Logstash server**因故障停止运行，数据也不会丢失，可靠性得到了大大的提升。

2）**用户轨迹跟踪**：kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等操作，这些活动信息被各个服务器发布到kafka的topic中，然后消费者通过订阅这些topic来做实时的监控分析，当然也可以保存到数据库。

3）**运营指标**：kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。

4）**流式处理**：比如spark streaming和storm。



**Kafka先持久化还是先同步副本？**

先持久化。



**Kafka中Zookeeper的作用？**

kafka集群中有一个broker会被选举成Controller，负责管理集群broker的上下线、所有topic的分区副本分配和leader选举。

Controller管理依赖于Zookeeper。



**topic 与 partition**

Topic-Partition为一对多。

topic 是逻辑上的概念，而partition 是物理上的概念，每个partition 对应一个log 文件，该log 文件中存储的就是producer 生产的数据。Producer 生产的数据会被不断追加到该log 文件末端，且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费。



**kafka 如何保证顺序消费**

要想实现消息有序，需要从Producer和Consumer两方面来考虑。

对于Producer，需要保证1个Topic只能对应1个Partition。

Consumer则需要使用单线程或者保证消费顺序的线程模型（RocketMQ 只需要在@RocketMQMessageListener注解加 consumeMode = ConsumeMode.ORDERLY 即可）。

如果只是想保证局部有序，只需要在发消息的时候指定Partition Key，Kafka对其进行Hash计算，根据计算结果决定放入哪个Partition。



**kafka 的消息可靠性如何保证**



**kafka 的 controller 选举和 leader 选举**



**Kafka如何广播消息？Consumer group**



**Kafka 如何保证数据高可用？副本，ack，HW**

request.required.acks有三个值0 1 -1(all)

**0 :** 生产者不会等待broker的ack，这个延迟最低但是存储的保证最弱当server挂掉的时候就会丢数据。

**1：**服务端会等待ack值leader副本确认接收到消息后发送ack但是如果leader挂掉后他不确保是否复制完成新leader也会导致数据丢失。

**-1(all)：**服务端会等所有的follower的副本受到数据后才会受到leader发出的ack，这样数据不会丢失。



**解释ISR、AR、HW、LEO、LSO、LW**

LEO：是 LogEndOffset 的简称，代表当前日志文件中下一条。

HW：水位或水印（watermark）一词，也可称为高水位(high watermark)，通常被用在流式处理领域（比如Apache Flink、Apache Spark等），以表征元素或事件在基于时间层面上的进度。在Kafka中，水位的概念反而与时间无关，而是与位置信息相关。严格来说，它表示的就是位置信息，即位移（offset）。取 partition 对应的 ISR中 最小的 LEO 作为 HW，consumer 最多只能消费到 HW 所在的位置上一条信息。

LSO：是 LastStableOffset 的简称，对未完成的事务而言，LSO 的值等于事务中第一条消息的位置(firstUnstableOffset)，对已完成的事务而言，它的值同 HW 相同。

LW：Low Watermark 低水位, 代表 AR 集合中最小的 logStartOffset 值。



**如何判断副本是否应该属于 ISR?** 

Follower 副本的 LEO 落后 Leader LEO 的时间，是否超过了 Broker 端参数 replica.lag.time.max.ms 值。如果超过了，副本就会被从 ISR 中移除。



**Producer 如何保证数据发送不丢失？**



**什么情况会触发follower副本的同步请求？什么时候向leader副本拉取数据？**



**什么情况副本会失效？是谁监听副本的失效？是谁负责从ISR中移除失效的副本？**



**producer向broker发送消息的流程？拦截器->序列化器->分区器是否了解？它们之间的处理顺序是什么？**



**producer怎么知道某个partition的leader是谁？**



**kafka的消息压缩算法能讲一下么？**



**消费者消费消息，自动提交offset，如果提交完后consumer宕机了，怎么搞？如果把自动提交关了，等consumer消费完了之后提交offset，当consumer消费业务处理完了之后宕机，此时还未提交offset怎么处理？**



**能说一下kafka中关于segment的概念么？**



**为什么kafka能保证这么高的吞吐量？[生产者批量发送/压缩]**

**顺序读写**：

**零拷贝**：

**文件分段**：segment。

**批量发送**：Producer端可以在内存中合并多条消息后，以一次请求的方式发送了批量的消息给broker，从而大大减少broker存储消息的IO操作次数。但也一定程度上影响了消息的实时性，相当于以时延代价，换取更好的吞吐量。

**数据压缩**：Producer端可以通过GZIP或Snappy格式对消息集合进行压缩。Producer端进行压缩之后，在Consumer端需进行解压。压缩的好处就是减少传输的数据量，减轻对网络传输的压力，在对大数据处理上，瓶颈往往体现在网络上而不是CPU（压缩和解压会耗掉部分CPU资源）。



**kafka为什么这么快(高性能)？[分区/顺序读写/pageCache/MMFile(Memory Mapped File)/分批发送/零拷贝/消息压缩/高性能二进制序列化/无锁offset管理]**

**kafka是如何保证顺序读写的？**



**ProducerRecord 中的时间戳是干嘛用的？CreateTime？LogAppendTime？**



**Leader副本所在节点会记录所有副本的LEO，Leader副本是什么时候记录follower副本的LEO的呢？**



**请讲一下leader与follower副本同步过程**



**讲一下Kafka消费者重平衡机制**



**什么时候会触发消费者ReBalance？[组消费者成员发生变化/订阅主题数发生变化/订阅主题的分区数发生变化]**



**ReBalance有什么问题么？在重平衡期间，消费者是否可用？**



**Kafka为什么不支持读写分离？[数据一致性问题/延时问题]**



**kafka发送消息的三种方式？[简单/同步/异步]**



**分区策略有哪些？如何自定义分区策略？**



**消费者组的意义？[reBalance/多应用全量消息]**



**两个订阅相同topic的消费者组，其中一个消费失败触发重试，另一个消费者组会不会重复消费？如何保证不会重复消费？**



**如果消费者组中的其中一个消费者挂了，会发生什么？**



**Consumer是单线程么？[pull/心跳]**



**Kafka 的 Consumer 客户端是线程安全的么？[不安全，单线程消费，多线程处理] 如何保证线程安全？[Reactor模型]**



**消费者提交消费位移时提交的是当前消费到的最新消息的offset还是offset+1？[offset+1]**



**kafka消息保留策略知道么？[时间/文件大小]**



**讲一下你对kafka流处理(聚合/统计)的理解？**



**目前kafka某主题8个分区，我能修改成16个么？我能修改成4个么？**



**zookeeper在kafka中的作用？[broker注册/topic注册/生产者负载均衡/消费者负载均衡/集群管理/元数据管理]**



**谁负责管理 flower 与 leader 之间的数据同步？[Broker内部ReplicaManager服务]**



**重试机制会不会影响kafka顺序？怎么解决？**



**讲一下Consumer群组协调器与再均衡监听器两个概念**



**Consumer自动提交会有什么问题？手动提交分为哪几种方式？[同步/异步/同步和异步/自定义提交]说一下自定义提交的理解**



**Consumer一次消费多条消息，但其中某条消费失败异常，这时候怎么保证不会消息丢失？顺便讲一下Kafka的事务处理**



**创建主题的时候，参数–replication-factor 能不能大于broker数量？[不能]**



**Producer和Consumer在send或者pull之前会先从broker中获取元数据，并且这些元数据会缓存在客户端，如果broker某台机器宕机，并重新选举leader后，这时候缓存的数据可能就不准确，这时候会出现什么问题？该怎么处理？[broker会返回异常]**



**知道kafka超时数据清理、数据压缩机制么？**



**不完全的Leader选举？ unclean.leader.election参数的作用是什么？**



**Kafka在哪处用到了零拷贝？页面缓存是什么意思？**



**如果controller所在的broker节点宕机了，zk是怎么知道并删除相关节点（broker/ids/controller）的？[Broker 与 ZK会话结束，znode自动删除]ZK又是怎么通知kafka的？[watch]**



**讲述FetchRequest RequestBody中session_id、epoch、forgotten_topics_data三个域的作用**