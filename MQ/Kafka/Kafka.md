# Kafka架构

![](D:\ainterview\interview\MQ\Kafka\img\5e7dd2835ac6476799987104e1c5194a~tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.webp)

Producer：消息生产者，向kafka broker发送消息

Consumer：消息消费者，从kafka broker取消息

Consumer Group：多个consumer组成，消费者组内不同消费者负责消费不同分区的数据，kafka的topic下的一条消息，只能被同一个消费者组的一个消费者消费到

- consumer group下可以有一个或多个consumer instance，consumer instance可以是一个进程，也可以是一个线程
- group.id是一个字符串，唯一标识一个consumer group
- **consumer group下订阅的topic下的每个分区只能分配给某个group下的一个consumer(当然该分区还可以被分配给其他group)**

Broker：一台服务器就是一个broker，一个集群由多个broker组成，一个broker可以容纳多个topic

Topic：主题（队列）

Partition：分区，kafka的扩展性体现，一个庞大的topic有很多分区（partition）,可以分不到多个broker上去，每个 partition 是一个有序的队列， partition 中的

每条消息 都会被分配一个有序的 id （offset） kafka 只保证按一个 partition 中的顺序将消息发给consumer ，不保证一个 topic的整体（多个 partition 间）的顺

序；

Replica：副本，当集群某个节点故障时，该节点的partitiion数据不丢失，kafka的副本机制，一个topic的每个分区有多个副本，一个leader和follower

follower：每个分区的多个副本的“从”，实时从leader中同步数据，保持leader数据的同步

leader：每个分区副本的主，生产者发送数据的对象，消费者消费数据的对象

Offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset,用

于partition唯一标识一条消息；kafka 的存储文件都是按照 offset.kafka来命名，用 offset 名字的好处是方便查。例如你想位于 2049，只要找到2048.kafka的文

件即可。当然 the first offsetthe 就 是 00000000000.kafka ；

zookeeper：保存offset数据（0.9版本之前），保证高可用，0.9版本之后offset数据存放在kafka的系统特定topic中；

## Producer

producer先从zookeeper的 "/brokers/.../state"节点找到该partition的leader。

producer将消息发送给该leader。

leader将消息写入本地log。

followers从leader pull消息。

写入本地log后向leader发送ACK。

leader收到所有ISR中的replication的ACK后，增加HW（high watermark，最后commit 的offset）并向producer发送ACK。



# 面试

## Kafka先持久化还是先同步副本？

## Kafka中Zookeeper的作用？

kafka集群中有一个broker会被选举成Controller，负责管理集群broker的上下线、所有topic的分区副本分配和leader选举。

Controller管理依赖于Zookeeper。

## topic 与 partition

topic 是逻辑上的概念，而partition 是物理上的概念，每个partition 对应一个log 文件，该log 文件中存储的就是producer 生产的数据。Producer 生产的数据会被不断追加到该log 文件末端，且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费。