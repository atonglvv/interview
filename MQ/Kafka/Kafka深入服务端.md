本章涉及协议设计、时间轮、延迟操作、控制器及参数解密,尤其是协议设计和控制器的介绍,这些是深入了解Kafka的必备知识点。

# 协议设计

Kafka自定义了一组基于TCP的二进制协议,只要遵守这组协议的格式,就可以向 Kafka发送消息,也可以从Kafka中拉取消息,或者做一些其他的事情,比如提交消费位移等。

在目前的Kafka2.0.0中,一共包含了43种协议类型,每种协议类型都有对应的请求(Request)和响应( Response),它们都遵守特定的协议模式。**每种类型的 Request都包含相同结构的协议请求头(RequestHeader)和不同结构的协议请求体(RequestBody)**,如图6-1所示。

![](.\img\协议设计01.png)		

协议请求头中包含4个域(Field): api_key、 api_version、 correlation_id和client _d,这4个域对应的描述可以参考表6-1。

![](.\img\协议设计02.png)	

每种类型的 Response也包含相同结构的协议响应头(ResponseHeader)和不同结构的响应体(ResponseBody),如图6-2所示。

![](.\img\协议设计03.png)	

协议响应头中只有一个 correlation_id,对应的释义可以参考表6-1中的相关描述。
细心的读者会发现不管是在图6-1中还是在图6-2中都有类似int32、int16、 string的字样,它们用来表示当前域的数据类型。 Kafka中所有协议类型的 Request和 Response的结构都是具备固定格式的,并且它们都构建于多种基本数据类型之上。这些基本数据类型如图6-2所示。

![](.\img\协议设计04.png)	

下面就以最常见的消息发送和消息拉取的两种协议类型做细致的讲解。首先要讲述的是消息发送的协议类型,即 ProduceRequest/ProduceResponse,对应的 api_key=0,表示 PRODUCE。从 Kafka建立之初,其所支持的协议类型就一直在增加,并且对特定的协议类型而言,内部的组织结构也并非一成不变。以 ProduceRequest/ProduceResponse为例,截至目前就经历了7个
版本(V0~V6)的变迁。下面就以最新版本(V6,即 api_version=6)的结构为例来做细致的讲解。 ProduceRequest的组织结构如图6-3所示。

![](.\img\协议设计05.png)	



除了请求头中的4个域,其余 ProduceRequest请求体中各个域的含义如表6-3所示。

![](.\img\协议设计06.png)	

在《Kafka生产者》中我们了解到:消息累加器RecordAccumulator中的消息是以<分区,DequeProducerBatch>的形式进行缓存的,之后由 Sender线程转变成<Node,List<ProducerBatch>>的形式,针对每个Node, Sender线程在发送消息前会将对应的List<ProducerBatch>形式的内容转变成 ProduceRequest的具体结构。List<ProducerBatch>中的内容首先会按照主题名称进行分类(对应 ProduceRequest中的域 topic),然后按照分区编号进行分类(对应 ProduceRequest中的域 partition),分类之后的 ProducerBatch集合就对应 ProduceRequest中的域 record_set从另一个角度来讲,每个分区中的消息是顺序追加的,那么在客户端中按照分区归纳好之后就可以省去在服务端中转换的操作了,这样将负载的压力分摊给了客户端,从而使服务端可以专注于它的分内之事,如此也可以提升整体的性能。

如果参数acks设置非0值,那么生产者客户端在发送 ProduceRequest请求之后就需要(异步)等待服务端的响应ProduceResponse。对 ProduceResponse而言,V6版本中 ProduceResponse的组织结构如图6-4所示。

![](.\img\协议设计07.png)	

除了响应头中的 correlation_id,其余 ProduceResponse各个域的含义如表6-4所示。

![](.\img\协议设计08.png)	

消息追加是针对单个分区而言的,那么响应也是针对分区粒度来进行划分的,这样ProduceRequest和 ProduceResponse做到了一一对应。

我们再来了解一下拉取消息的协议类型,即 FetchRequest/FetchResponse,对应的 api_key=1,表示 FETCH。截至目前,FetchRequest/FetchResponse一共历经了9个版本(V0~V8)的变迁,下面就以最新版本(V8)的结构为例来做细致的讲解。 FetchRequest的组织结构如图6-5所示。

![](.\img\协议设计09.png)	



除了请求头中的4个域,其余 FetchRequest中各个域的含义如表6-5所示。

![](.\img\协议设计10.png)	

不管是 follower副本还是普通的消费者客户端,如果要拉取某个分区中的消息,就需要指定详细的拉取信息,也就是需要设定 partition、 fetch_offset、log_start_offset和 max_bytes这4个域的具体值,那么对每个分区而言,就需要占用4B+8B+8B+4B=24B的
空间。一般情况下,不管是 follower副本还是普通的消费者,它们的订阅信息是长期固定的也就是说, FetchRequest中的 topics域的内容是长期固定的,只有在拉取开始时或发生某些异常时会有所变动。 FetchRequest请求是一个非常频繁的请求,如果要拉取的分区数有很多,比如有1000个分区,那么在网络上频繁交互 FetchRequest时就会有固定的1000×24B≈24KB的字节的内容在传动,如果可以将这24KB的状态保存起来,那么就可以节省这部分所占用的带宽。

Kafka从1.1.0版本开始针对 FetchRequest引入了 session_id、 epoch和 forgotten_topics_data等域, session_id和 epoch确定一条拉取链路的 fetch session,当 session建立或变更时会发送全量式的 FetchRequest,所谓的全量式就是指请求体中包含所有需要拉取的分区信息;当 session稳定时则会发送增量式的 FetchRequest请求,里面的 topics域为空,因为 topics域的内容已经被缓存在了 session链路的两侧。如果需要从当前 fetch session中取消对某些分区的拉取订阅,则可以使用 forgotten topics data字段来实现。

这个改进在大规模(有大量的分区副本需要及时同步)的 Kafka集群中非常有用,它可以提升集群间的网络带宽的有效使用率。不过对客户端而言效果不是那么明显,一般情况下单个客户端不会订阅太多的分区,不过总体上这也是一个很好的优化改进。与 FetchRequest对应的 FetchResponse的组织结构(V8版本)可以参考图6-6。

![](.\img\协议设计11.png)	



FetchResponse结构中的域也很多,它主要分为4层,第1层包含 throttle_time_ms、error_code、 session_id和 responses,前面3个域都见过,其中 session_id和FetchRequest中的 session_id对应。 responses是一个数组类型,表示响应的具体内容,也就是 FetchResponse结构中的第2层,具体地细化到每个分区的响应。第3层中包含分区的元数据信息( partition、 error_code等)及具体的消息内容(record_set)，aborted_transactions和事务相关。

除了 Kafka客户端开发人员,绝大多数的其他开发人员基本接触不到或不需要接触具体的协议,那么我们为什么还要了解它们呢?其实,协议的具体定义可以让我们从另一个角度来了解Kafka的本质。以 PRODUCE和 FETCH为例,从协议结构中就可以看出消息的写入和拉取消费都是细化到每一个分区层级的。并且,通过了解各个协议版本变迁的细节也能够从侧面了解 Kafka变迁的历史,在变迁的过程中遇到过哪方面的瓶颈,又采取哪种优化手段,比如FetchRequest中的 session_id的引入。

由于篇幅限制,笔者并不打算列出所有 Kafka协议类型的细节。不过对于Kafka协议的介绍并没有到此为止,后面的章节中会针对其余41种类型的部分协议进行相关的介绍,完整的协议类型列表可以参考官方文档1。 Kafka中最枯燥的莫过于它的上百个参数、几百个监控指标和几十种请求协议,掌握这三者的“套路”,相信你会对 Kafka有更深入的理解。



# 时间轮

Kafka中存在大量的延时操作,比如延时生产、延时拉取和延时删除等。 Kafka并没有使用JDK自带的 Timer或 DelayQueue来实现延时的功能,而是基于时间轮的概念自定义实现了一个用于延时功能的定时器(SystemTimer)。JDK中 Timer和 DelayQueue的插入和删除操作的平均时间复杂度为O(nlogn)并不能满足Kafka的高性能要求,而基于时间轮可以将插入和删除操作的时间复杂度都降为O(1)。时间轮的应用并非 Kafka独有,其应用场景还有很多,在Netty、Akka、 Quartz、 ZooKeeper等组件中都存在时间轮的踪影。

如图6-7所示, Kafka中的时间轮(TimingWheel)是一个存储定时任务的环形队列,底层采用数组实现,数组中的每个元素可以存放一个定时任务列表( TimerTaskList)。 TimerTaskList是一个环形的双向链表,链表中的每一项表示的都是定时任务项(TimerEntry),其中封装了真正的定时任务(TimerTask)。

时间轮由多个时间格组成,每个时间格代表当前时间轮的基本时间跨度(tickMs)。时间轮的时间格个数是固定的,可用 wheelSize来表示,那么整个时间轮的总体时间跨度(interval)可以通过公式 tickMs × wheelSize计算得出。时间轮还有一个表盘指针(currentTime),用来表示时间轮当前所处的时间, currentTime是 tickMs的整数倍。 currentTime可以将整个时间轮划分为到期部分和未到期部分, currentTime当前指向的时间格也属于到期部分,表示刚好到期,需要处理此时间格所对应的 TimerTaskList中的所有任务。