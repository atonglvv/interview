# Kafka架构

## 核心概念

![](..\Kafka\img\架构图.webp)

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

序，分区在存储层面可看做是一个可追加的日志文件。

Replica：副本，当集群某个节点故障时，该节点的partitiion数据不丢失，kafka的副本机制，一个topic的每个分区有多个副本，一个leader和follower

follower：每个分区的多个副本的“从”，实时从leader中同步数据，保持leader数据的同步

leader：每个分区副本的主，生产者发送数据的对象，消费者消费数据的对象

Offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset,用

于partition唯一标识一条消息；kafka 的存储文件都是按照 offset.kafka来命名，用 offset 名字的好处是方便查。例如你想位于 2049，只要找到2048.kafka的文

件即可。当然 the first offsetthe 就 是 00000000000.kafka ；

zookeeper：保存offset数据（0.9版本之前），保证高可用，0.9版本之后offset数据存放在kafka的系统特定topic中；

## 多副本机制

一个分区会在多个副本中保存相同的消息。

副本之间是一主多从关系。

leader副本负责读写操作，follower副本只负责同步消息（主动拉取）。

leader副本故障时，从follower副本重新选举新leader。

![](..\Kafka\img\多副本机制.jpg)	

分区中所有副本统称为 **AR（Assigned Replicas）**

所有与leader副本保持一定程度同步的副本（包括leader）组成 **ISR（In-Sync Replicas）**

同步滞后过多的副本组成 **OSR（Out-of-Sync Replicas）**

## 分区的偏移量

![](..\Kafka\img\分区偏移量.jpg)	

LEO（Log End Offset）：标识当前分区下一条代写入消息的offset。

HW（High Watermark）：高水位，标识了一个特定的offset，消费者只能拉取到这个offset之前的消息（不含HW）。

分区ISR集合中的每个副本都会维护自身的LEO，而ISR集合中最小的LEO即为分区的HW，对消费者而言只能消费HW之前的消息。

## Producer

producer先从zookeeper的 "/brokers/.../state"节点找到该partition的leader。

producer将消息发送给该leader。

leader将消息写入本地log。

followers从leader pull消息。

写入本地log后向leader发送ACK。

leader收到所有ISR中的replication的ACK后，增加HW（high watermark，最后commit 的offset）并向producer发送ACK。

### 必要参数配置：

bootstrap.servers：设置kafka集群地址，并非需要所有broker地址，因为生产者会从给定的broker中获取其他broker信息

key.serializer、value.serializer：转换字节数组到所需对象的序列化器，填写全限类名

### 发送模式

发后即忘（fire-and-forget）：只管往kafka发送而不关心消息是否正确到达，不对发送结果进行判断处理；

同步（sync）：KafkaProducer.send()返回的是一个Future对象，使用Future.get()来阻塞获取任务发送的结果，来对发送结果进行相应的处理；

异步（async）：向send()返回的Future对象注册一个Callback回调函数，来实现异步的发送确认逻辑。

### 生产者拦截器

实现ProducerInterceptor接口，在消息发送的不同阶段调用

configure()：完成生产者配置时

onSend()：调用send()后，消息序列化和计算分区之前

onAcknowledgement()：消息被应答之前或消息发送失败时

close()：生产者关闭时

通过 interceptor.classes 配置指定

### 序列化

自定义序列化器：实现Serializer接口

### 分区器

在消息发送到kafka前，需要先计算出分区号，默认使用DefaultPartitioner（采用MurmurHash2算法）

自定义分区器：实现Partitioner接口

通过partitioner.class配置指定

### 生产者客户端的整体架构

![](..\Kafka\img\生产者客户端的整体架构.jpg)	

主线程KafkaProducer创建消息，通过可能的拦截器、序列化器和分区器之后缓存到消息累加器（RecordAccumulatro）。

消息在RecordAccumulator被包装成ProducerBatch，以便Sender线程可以批量发送，缓存的消息发送过慢时，send()方法会被阻塞或抛异常。

缓存的大小通过buffer.memory配置，阻塞时间通过max.block.ms配置。

Kafka生产者客户端中，通过ByteBuffer实现消息内存的创建和释放，而RecordAccumulator内部有一个BufferPool用来实现ByteBuffer的复用。

Sender从RecordAccumulator中获取缓存的消息后，将ProducerBatch按Node分组，Node代表broker节点。也就是说sender只向具体broker节点发送消息，而

不关注属于哪个分区，这里是应用逻辑层面到网络层面的转换。

Sender发往Kafka前，还会保存到InFlightRequests中，其主要作用是缓存已经发出去但还没收到相应的请求，也是以Node分组。

每个连接最大缓存未响应的请求数通过max.in.flight.requests.per.connection配置(默认5)。

### 元数据的更新

InFlightRequests可以获得leastLoadedNode，即所有Node中负载最小的。leastLoadedNode一般用于元数据请求、消费者组播协议等交互。

当客户端中没有需要使用的元数据信息或超过metadata.max.age.ms没有更新元数据时，就会 引起元数据更新操作。

### 重要的生产者参数

acks：用来指定分区中有多少个副本收到这条消息，生产者才认为写入成功（默认”1"）

“1"：leader写入即成功、“0”：不需要等待服务端相应、”-1”/“all"：ISR所有副本都写入才收到响应

max.request.size：限制生产者客户端能发送的消息的最大值（默认1048576，即1m）

retries、retry.backoff.ms：生产者重试次数（默认0）和两次重试之间的间隔（默认100）

compression.type：消息压缩方式，可配置为”gzip”、”snappy”、”lz4”（默认”none”）

connections.max.idle.ms：多久后关闭闲置的连接（默认540000，9分钟）

linger.ms：生产者发送ProducerBatch等待更多消息加入的时间（默认为0）

receive.buffer.bytes：Socket接收消息缓冲区的大小（默认32768，32k）

send.buffer.bytes：Socket发送消息缓冲区的大小（默认131072，128k）

request.timeout.ms：Producer等待请求响应的最长时间（默认30000ms），这个值需要比broker参数replica.lag.time.max.ms大

## Consumer

### 消费者与消费者组

每个分区只能被一个消费组的一个消费者消费。

消费者数大于分区数时，会有消费者分配不到分区而无法消费任何消息。

消费者并非逻辑上的概念，它是实际的应用实例，它可以是一个钱程，也可以是一个进程。

传统的消息引擎处理模型主要有两种，队列模型，和发布-订阅模型：

- 队列模型：早期消息处理引擎就是按照队列模型设计的，所谓队列模型，跟队列数据结构类似，生产者产生消息，就是入队，消费者接收消息就是出队，并删除队列中数据，消息只能被消费一次。 但这种模型有一个问题，那就是只能由一个消费者消费，无法直接让多个消费者消费数据。基于这个缺陷，后面又演化出发布-订阅模型。

- 发布-订阅模型： 发布订阅模型中，多了一个主题。消费者会预先订阅主题，生产者写入消息到主题中，只有订阅了该主题的消费者才能获取到消息。这样一来就可以让多个消费者消费数据。

消费组是一个逻辑上的概念，它将旗下的消费者归为一类，每一个消费者只隶属于一个消费组。每一个消费组都会有一个固定的名称，消费者在进行消费前需要指

定其所属消费组的名称，这个可以通过消费者客户端参数 **group.id** 来配置，默认值为空字符串。

### 客户端开发

一个正常的消费逻辑需要具有如下几个步骤：

（1）配置消费者客户端参数及创建相应的消费者实例

（2）订阅主题

（3）拉取消息并消费

（4）提交消费位移

（5）关闭消费者实例

#### 必要的参数配置

- bootstrap.servers ：该参数的释义和生产者客户端 KafkaProducer 中 的相同，用来指定连接 Kafka 集群所需的 broker 地址清单，具体内容形式为host1:port1,host2 : port2 ，可以设置一个或多个地址，中间用逗号隔开，这里设置两个以上的 broker 地址信息，当其中任意一个看机时，消费者仍然可以连接到 Kafka 集群上。
- group.id ：消费者隶属的消费组的名称 ， 默认值为“”。 如果设置为空，则会报出异
  常 ： Exception in thread "main” org.apache.kafka.common.errors.InvalidGroupldException:
  The configured groupld is invalid 。 一般而言，这个参数需要设置成具有一定的业务意义
  的名称 。
- key.deserializer 和 value.deserializer ：与生产者客户端 KafkaProducer中的 key.serializer 和 value.serializer 参数对应。消费者从 broker 端获取的消息格式都是字节数组( byte[]）类型，所以需要执行相应的反序列化操作才能还原成原有的对象格式 。

#### 订阅主题与分区

一个消费者可以订阅一个或多个主题。使用subscribe()方法，以集合的形式或者是正则的形式订阅特定的主题。subscribe()重载方法如下：

```java public void subscribe(Collection<String> topics ,ConsumerRebalanceListener listener)
public void subscribe(Collection<String> topics)
public void subscribe (Pattern pattern , ConsumerRebalanceListener listener)
public void subscribe (Pattern pattern)    
```

消费者不仅可以订阅主题，还可以直接订阅主题的特定分区，KafkaConsumer提供了一个assign()方法实现这功能：

```java
public void assign(Collection<TopicPartition> partitions)
```

#### 反序列化

与KafkaProducer的序列化器对应，可以通过实现Deserializer<T>自定义序列化器。

#### 消息消费

 **Kafka的消费是基于拉模式的。**Kafka中的消息消费是一个不断轮询的过程，消费者重复调用poll()方法，返回的是订阅主题（分区）上的一组消息。

```java
public class ConsumerRecord<K, V> {
    private final String topic;
    private final int partition;
    private final long offset ;
    private final long timestamp;
    private final TimestampType timestampType;
    private final int serializedKeySize;
    private final int serializedValueSize;
    private final Headers headers;
    private final K key ;
    private final V value ;
    private volatile Long checksum;
    //省略若干方法
}
```

offset 表示消息在所属分区的偏移量； timestamp 表示时间戳； key 和 value 分别表示消息的键和消息的值，一般业务应用要读取的就是value 。

#### 位移提交

对于分区中的每条消息都有唯一的offset（偏移量），用来表示消息在分区中对应的位置。对于消费者而言，也有一个offset（位移），消费者使用offset表示消费到分区中某个消息所在的位置。

每次调用poll()方法时，返回的是还没有被消费过的消息集，所以需要记录上一次消费时的位移，并且这个消费位移必须持久化存储。新版kafka中，消费位移存储在Kafka内部主题为 _consumer_offset 的topic中，这种把消费位移存储的动作称为"提交"，**消费者在消费完消息之后需要再执行消费位移的提交**。

Kafka默认的消费位移的提交方式是自动提交，由消费者客户端参数enable.auto.commit配置，默认值为true，自动提交并不是每消费一条消息就提交一次，而是定期提交，提交周期由客户端参数auto.commit.interval.ms配置，默认5秒。

Kafka位移提交是一个难点，存在重复消费和消息丢失的问题。

Kafka还提供了手动位移提交的方式，使得开发人员对消费位移的管理控制更灵活。开启手动提交功能的前提是enable.auto.commit配置为false。

手动提交又可分为**同步提交**与**异步提交**，对应KafkaConsumer的commitSync()和commitAsync()两种方法。

#### 控制或关闭消费

  Kafka提供了对消费速度进行控制的方法，某些应用场景下我们需要暂停某些分区的消费而先消费其他分区，当达到一定条件再恢复这些分区的消费。KafkaConsumer使用**pause()**和**resume()**方法分别实现**暂停**某些分区在拉取操作时返回数据给客户端和**恢复**某些分区向客户端返回数据的操作。

```java
public void pause(Collection<TopicPartition> partitions)
public void resume(Collection<TopicPartition< partitions)
```

#### 指定位移消费

有了消费位移的持久化才使消费者在崩溃或再平衡的时候，可以让接替的消费者能根据存储的位移维持继续消费。试想，当一个新的消费组建立的时候，它根本没有可以查找的消费位移，或者消费组的一个新消费者订阅了一个新的主题，它也没有可以查找的消费位移。

在Kafka中每当消费者查找不到所消费的消费位移时，就会根据消费者客户端参数auto.offset.rest的配置决定从何处开始消费，这个参数的三个值：

- "latest"，默认值，表示从分区末尾开始消费。
- "earliest"，表示从起始处开始消费。
- "none"，表示当出现查找不到消费位移的时候，抛出NoOffstForPartitionException。

有些时候，我们需要可以让我们从特定的位移处开始拉取消息，KafkaConsumer中的seek()方法提供了这个功能，让我们可以追前消费或回溯消费。

```java
Public void seek(TopicPartition partition, long offset)
```

#### 再平衡

当有新的消费者加入消费者组、已有的消费者退出消费者组或者所订阅的主题的分区发生变化，就会触发到分区的重新分配，重新分配的过程叫做再平衡

（Rebalance）。在再平衡发生的这一小段时间内，消费组变得不可用。

前面讲subscribe()方法的说到了再平衡监听器ConsumerRebalanceListener，再平衡监听器可以用来设定发生再平衡动作前后的一些准备或收尾动作（比如发生

再平衡前提交offset等。）

#### 消费者拦截器

 消费者拦截器主要是消费到消息或提交消息位移时进行一些操作。需要自定义实现org.apache.kafka.clients.consumer.ConsumerInterceptor接口。

### 重要的消费者参数

- fetch.min.bytes：一次请求能拉取的最小数据量（默认1b）
- fetch.max.bytes：一次请求能拉取的最大数据量（默认52428800b，50m）
- fetch.max.wait.ms：与min.bytes有关，指定kafka拉取时的等待时间（默认500ms）
- max.partition.fetch.bytes：从每个分区里返回Consumer的最大数据量（默认1048576b，1m）
- max.poll.records：一次请求拉取的最大消息数（默认500）
- connections.max.idle.ms：多久后关闭闲置连接，默认（540000，9分钟）
- receive.buffer.bytes：Socket接收消息缓冲区的大小（默认65536，64k）
- send.buffer.bytes：Socket发送消息缓冲区的大小（默认131072，128k）
- request.timeout.ms：Consumer等待请求响应的最长时间（默认30000ms）
- metadata.max.age.ms：元数据过期时间（默认30000，5分钟）
- reconnect.backoff.ms：尝试重新连接指定主机前的等待时间（默认50ms）
- retry.backoff.ms：尝试重新发送失败请求到指定主题分区的等待时间（默认100ms）
- isolation.level：消费者的事务隔离级别（具体查看进阶篇：事务）

## 主题与分区

分区的划分不仅可以为Kafka提供了可伸缩性，水平扩展能力，还可以通过副本机制来为Kafka提供数据冗余以提高数据的可靠性，为了做到均匀分布，通常partition的数量通常是BrokerServer数量的整数倍。

**主题、分区、副本、日志关系：**

![](..\Kafka\img\主题-分区-副本.png)	

### 分区

- 每个topic（逻辑名称）由一个或多个分区组成，分区是topic物理上的分组，在创建topic时被指定。

- 一个partition只对应一个Broke，一个Broke可以管理多个partition。

- 由消息在顺序写入，在同一个分区内的消息是有序的，在不同的分区间，kafka并不保证消息的顺序（所以kafka消息是支持跨分区的）。

- 同一个主题下，不同分区所包含的内容是不同的，每个消息被添加到分区当中时，会被分配一个偏移量（Offset），它是消息在分区当中的唯一编号，kafka通

  过offset来确保分区内的消息是顺序的，offset的顺序并不跨越分区。

- 要想保证消息顺序消息，需要将partion数目设为1。

#### 分区写入策略

所谓分区写入策略，即是生产者将数据写入到kafka主题后，kafka如何将数据分配到不同分区中的策略。常见的有三种策略：

- 随机策略：每次都随机地将消息分配到每个分区。
- 轮询策略：按顺序轮流将每条数据分配到每个分区中。轮询策略是默认的策略，除非有特殊的业务需求，否则使用这种方式即可。
- 按键（key）保存策略：当生产者发送数据的时候，可以指定一个key，计算这个key的hashCode值，按照hashCode的值对不同消息进行存储。

kafka默认是实现了两个策略，没指定key的时候就是轮询策略，有的话那就是按键保存策略。

#### 实现自定义分区

Kafka提供了两种让我们自己选择分区的方法：

- 第一种是在发送producer的时候，在ProducerRecord中直接指定，但需要知道具体发送的分区index，所以并不推荐。
-  第二种则是需要实现Partitioner.class类，并重写类中的partition(String topic, Object key, byte[] keyBytes,Object value, byte[] valueBytes, Cluster cluster) 方法，后面在生成kafka producer客户端的时候直接指定新的分区类就可以了。

### 副本

在kafka中，每个主题可以有多个分区，每个分区又可以有多个副本。这多个副本中，只有一个是leader，而其他的都是follower副本。仅有leader副本可以对外提

供服务。多个follower副本通常存放在和leader副本不同的broker中，通过这样的机制实现了高可用，当某台机器挂掉后，其他follower副本也能迅速”转正“，开

始对外提供服务。

#### AR ISR OSR

**AR**：分区中的所有副本统称为AR ( Assigned Replicas ) 。

**ISR**：所有与leader 副本保持一定程度同步的副本（包括leader 副本在内）组成ISR （In-Sync Replicas ) 。所谓“ 一定程度的同步”是指可忍受的滞后范围，这个范围可以通过参数进行配置。

**OSR**: 与leader 副本同步滞后过多的副本（不包括leader 副本）组成OSR ( Out-of-Sync Replicas ），由此可见， AR=ISR+OSR 。

默认情况下， 当leader 副本发生故障时，只有在ISR集合中的副本才有资格被选举为新的leader， 而在OSR集合中的副本则没有任何机会（不过这个原则也可以通过修改相应的参数配置来改变） 。

#### Kafka的副本有哪些作用

在kafka中，副本的目的就是冗余备份，且仅仅是冗余备份，所有的读写请求都是由leader副本进行处理的。follower副本仅有一个功能，那就是从leader副本拉取消息，尽量让自己跟leader副本的内容一致。

#### 为什么follow副本不对外提供服务

如果follower副本也对外提供服务那会怎么样呢？首先，性能是肯定会有所提升的。但同时，会出现一系列一致性问题。类似数据库事务中的幻读，脏读。比如你现在写入一条数据到kafka主题a，消费者b从主题a消费数据，却发现消费不到，因为消费者b去读取的那个分区副本中，最新消息还没写入。而这个时候，另一个消费者c却可以消费到最新那条数据，因为它消费了leader副本。

## 日志存储

### 文件目录布局

不考虑多副本的情况， 一个分区对应一个日志（ Log）。为了防止 Log 过大，Kafka 又引入了日志分段（ LogSegment ）的概念， 将 Log 切分为多个 LogSegment，相当于一个巨型文件被平均分配为多个相对较小的文件，这样也便于消息的维护和清理。

事实上， Log 和LogSegment也不是纯粹物理意义上的概念， **Log 在物理上只以文件夹的形式存储**，而**每个LogSegment 对应于磁盘上的一个日志文件和两个索引文件，以及可能的其他文件（比如以“ .txnindex ”为后缀的事务索引文件）**。 

Log对应了一个命名形式为<topic>-<partition>的文件夹。举个例子，假设有一个名为"topic-log"的topic，此topic中有4个分区，那么在实际物理存储上表现为"topic-log-0" "topic-log-1" "topic-log-2" "topic-log-3" 这4个文件夹。

![](..\Kafka\img\日志.png)	  

向 Log 中追加消息时是顺序写入的，只有最后一个 LogSegment 才能执行写入操作，在此之前所有的 LogSegment 都不能写入数据。

为了便于消息的检索，每个 LogSegment 中的日志文件（以“ .log”为文件后缀）都有对应的两个索引文件 ：偏移量索引文件（以“ .index”为文件后缀）和时间戳索引文件（以“ .timeindex ”为文件后缀）。

每个 LogSegment 都有一个基准偏移量 baseOffset，用来表示当前 LogSegment中第一条消息的 offset 。 偏移量是一个 64 位的长整型数，日志文件和两个索引文件都是根据基准偏移量（ baseOffset）命名的，名称固定为 20 位数字，没有达到的位数则用 0 填充 。 比如第一个 LogSegment 的基准偏移量为 0，对应的日志文件为 00000000000000000000 .logo

注意每个LogSegment中不只包含".log" ".index" ".timeindex" 这三种文件，还可能包含".deleted" ".cleaned" ".swap" 等临时文件，以及可能的".snapshot" ".txnindex" "leader-epoch-checkpoint" 等文件。

### 数据压缩

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

### 日志索引

偏移量索引文件用来建立消息偏移量（offset）到物理地址之间的映射关系，方便快速定位消息所在的物理文件位置；时间戳索引文件则根据指定的时间戳来查找对应的偏移量信息。

Kafka中的索引文件以稀疏索引的方式添加索引项，每当写入一定量（由broker端参数log.index.interval.bytes指定，默认为4k）的消息时，就会在偏移量索引文件和时间戳索引文件中增加一个索引项。

Kafka通过**MappedByteBuffer**将索引文件映射到内存中，来加快索引的查询速度。

#### 偏移量索引

对于偏移量索引文件，保存的是 **<相对偏移量，物理地址>** 的对应关系，文件中的相对偏移量是单调递增的。

查询**指定偏移量**对应的消息时，使用**改进的二分查找算法**来快速定位偏移量的位置，如果指定的偏移量不在索引文件中，则会返回文件中小于指定偏移量的最大偏移量及对应的物理地址，该逻辑通过OffsetIndex.lookup()方法实现。

偏移量索引文件的索引项结构如下图所示，每个索引项记录了相对偏移量relativeOffset和对应消息的第一个字节在日志段文件中的物理地址position，共占用8个字节。

![](..\Kafka\img\偏移量索引项.png)	

为什么使用相对偏移量？这样可以节约存储空间。每条消息的绝对偏移量占用8个字节，而相对偏移量只占用4个字节（relativeOffset=offset-baseOffset）。在日志段文件滚动的条件中，有一个是：追加消息的最大偏移量和当前日志段的baseOffset的差值大于Int.MaxValue，因为如果大于这个值，4个字节就无法存储相对偏移量了。

##### 偏移量索引文件的查找原理

假设要查找偏移量为230的消息，查找过程如下：

![](..\Kafka\img\偏移量索引文件的查找原理.png)	

首先找到baseOffset=217的日志段文件（这里使用了跳跃表的结构来加速查找）。

计算相对偏移量relativeOffset=230-217=13。

在索引文件中查找不大于13的最大相对偏移量对应的索引项，即[12,456]。

根据12对应的物理地址456，在日志文件.log中定位到准确位置。

从日志文件物理位置456继续向后查找找到相对偏移量为13，即绝对偏移量为230，物理地址为468的消息。

注意：

消息在log文件中是以批次存储的，而不是单条消息进行存储。索引文件中的偏移量保存的是该批次消息的最大偏移量，而不是最小的。

Kafka强制要求索引文件大小必须是索引项大小（8B）的整数倍，假设broker端参数log.index.size.max.bytes设置的是67，那么Kafka内部也会将其转为64，即不大于67的8的最大整数倍。

#### 时间戳索引

对于时间戳索引文件，保存的是 **<时间戳，相对偏移量>** 的对应关系，文件中的时间戳和相对偏移量都是单调递增的。

查询**指定时间戳**对应的消息时， 需要配合偏移量索引文件进行查找。首先通过改进的二分查找在时间戳索引文件中找到不大于目标时间戳的索引项，然后根据索引项的相对偏移量在偏移量索引文件中查找，查找方式就是上面指定偏移量的方式。 

时间戳索引文件的索引项结构如下图所示，每个索引项记录了时间戳timestamp和相对偏移量relativeOffset的对应关系，共占用12个字节。

![](..\Kafka\img\时间戳索引项.png)	

##### 时间戳索引文件的查找原理

假设要查找时间戳为1540的消息，查找过程如下（这里时间戳只是一个示意值）：

![](..\Kafka\img\时间戳索引文件的查找原理.png)	

将要查找的时间戳1540和每个日志段的最大时间戳逐一对比，直到找到最大时间戳不小于1540的日志段。（日志段的最大时间戳：获取时间戳索引文件最后一个索引项的时间戳，如果大于0，取该值；否则取日志段的最近修改时间）

找到对应的日志段后，在时间戳索引文件中使用二分查找找到不大于目标时间戳1540的最大索引项，即图中的[1530,12]，获取对应的相对偏移量12

在该日志段的偏移量索引文件中找到相对偏移量不大于12的索引项，即图中的[12，456]

在日志文件中从物理位置456开始查找时间戳不小于1540的消息

**注意：**

Kafka强制要求索引文件大小必须是索引项大小（12B）的整数倍，假设broker端参数log.index.size.max.bytes设置的是67，那么Kafka内部也会将其转为60，即不大于67的12的最大整数倍。

虽然写数据时偏移量索引文件和时间戳索引文件会同时写入一个索引项，但是两个索引项的相对偏移量不一定是一样的，这是因为：生产者生产消息时可以指定时间戳，导致一个批次中的消息，偏移量最大的对应的时间戳不一定最大，而时间戳索引文件中保存的是一个批次中最大的时间戳及对应消息的相对偏移量。

这里查找目标时间戳对应的日志段时，就无法采用跳表来快速查找了，好在日志段的最大时间戳是递增的，依次查看就行了。至于为什么不单独写一个数据结构保存最大时间戳和日志段对象的对应关系，大概是通过时间戳查找消息的操作用的很少吧。

### 日志清理

Kafka提供了两种日志清理策略：

- 日志删除（Log Retention）：按照一定的保留策略**直接删除**不符合条件的日志分段。
- 日志压缩（Log Compaction）：针对消息的key进行整合，对于有相同key的但有不同value值，只保留最后一个版本。

#### 日志删除

在 Kafka 的日志管理器中会有一个专门的日志删除任务来周期性地检测和删除不符合保留条件的日志分段文件，这个周期可以通过 broker 端参数 `log.retention.check.interval.ms` 来配置，默认值为300000，即5分钟。当前日志分段的保留策略有3种：

- 基于时间的保留策略
- 基于日志大小的保留策略
- 基于日志起始偏移量的保留策略。

##### 基于时间

日志删除任务会检查当前日志文件中是否有保留时间超过设定的阈值来寻找可删除的日志分段文件集合，如下图所示。阈值可以通过 broker 端参数 `log.retention.hours`、`log.retention.minutes` 和 `log.retention.ms` 来配置，其中 `log.retention.ms` 的优先级最高，`log.retention.minutes` 次之，`log.retention.hours` 最低。默认情况下只配置了 `log.retention.hours` 参数，其值为168，故默认情况下日志分段文件的保留时间为7天。

![](..\Kafka\img\日志删除-基于时间.png)	

查找过期的日志分段文件，并不是简单地根据日志分段的最近修改时间 lastModifiedTime 来计算的，而是根**据日志分段中最大的时间戳 largestTimeStamp 来计算**的。因为日志分段的 lastModifiedTime 可以被有意或无意地修改，比如执行了 touch 操作，或者分区副本进行了重新分配，lastModifiedTime 并不能真实地反映出日志分段在磁盘的保留时间。

要获取日志分段中的最大时间戳 largestTimeStamp 的值，首先要查询该日志分段所对应的时间戳索引文件，查找时间戳索引文件中最后一条索引项，若最后一条索引项的时间戳字段值大于0，则取其值，否则才设置为最近修改时间 lastModifiedTime。

##### 基于日志起始偏移量

一般情况下，日志文件的起始偏移量 logStartOffset 等于第一个日志分段的 baseOffset，但这并不是绝对的，logStartOffset 的值可以通过 DeleteRecordsRequest 请求（比如使用 KafkaAdminClient 的 deleteRecords() 方法、使用 kafka-delete-records.sh 脚本）、日志的清理和截断等操作进行修改。

![](..\Kafka\img\日志删除-基于日志起始偏移量.png)	

基于日志起始偏移量的保留策略的判断依据是某日志分段的下一个日志分段的起始偏移量 baseOffset 是否小于等于 logStartOffset，若是，则可以删除此日志分段。如上图所示，假设 logStartOffset 等于25，日志分段1的起始偏移量为0，日志分段2的起始偏移量为11，日志分段3的起始偏移量为23，通过如下动作收集可删除的日志分段的文件集合 deletableSegments：

- 从头开始遍历每个日志分段，日志分段1的下一个日志分段的起始偏移量为11，小于 logStartOffset 的大小，将日志分段1加入 deletableSegments。
- 日志分段2的下一个日志偏移量的起始偏移量为23，也小于 logStartOffset 的大小，将日志分段2加入 deletableSegments。
- 日志分段3的下一个日志偏移量在 logStartOffset 的右侧，故从日志分段3开始的所有日志分段都不会加入 deletableSegments。

收集完可删除的日志分段的文件集合之后的删除操作同基于日志大小的保留策略和基于时间的保留策略相同。

##### 基于日志大小

日志删除任务会检查当前日志的大小是否超过设定的阈值来寻找可删除的日志分段的文件集合，如下图所示。阈值可以通过 broker 端参数 `log.retention.bytes` 来配置，默认值为-1，表示无穷大。注意 `log.retention.bytes` 配置的是 Log 中所有日志文件的总大小，而不是单个日志分段（确切地说应该为 .log 日志文件）的大小。单个日志分段的大小由 broker 端参数 `log.segment.bytes `来限制，默认值为1073741824，即 1GB。

![](..\Kafka\img\日志删除-基于日志大小.png)		

基于日志大小的保留策略与基于时间的保留策略类似，首先计算日志文件的总大小 size 和阈值的差值 diff，即计算需要删除的日志总大小，然后从日志文件中的第一个日志分段开始进行查找可删除的日志分段的文件集合。查找出它之后就执行删除操作，这个删除操作和基于时间的保留策略的删除操作相同。

#### 日志压缩

# 面试

## Kafka先持久化还是先同步副本？

## Kafka中Zookeeper的作用？

kafka集群中有一个broker会被选举成Controller，负责管理集群broker的上下线、所有topic的分区副本分配和leader选举。

Controller管理依赖于Zookeeper。

## topic 与 partition

Topic-Partition为一对多。

topic 是逻辑上的概念，而partition 是物理上的概念，每个partition 对应一个log 文件，该log 文件中存储的就是producer 生产的数据。Producer 生产的数据会被不断追加到该log 文件末端，且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费。