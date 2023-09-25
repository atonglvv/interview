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

![](.\img\时间轮01.png)	

若时间轮的 tickMs为1ms且 wheelSize等于20,那么可以计算得出总体时间跨度 interval为20ms。初始情况下表盘指针currentTime指向时间格0,此时有一个定时为2ms的任务插进来会存放到时间格为2的 TimerTaskList中。随着时间的不断推移,指针 currentTime不断向前推进,过了2ms之后,当到达时间格2时,就需要将时间格2对应的 TimeTaskList中的任务进行相应的到期操作。此时若又有一个定时为8ms的任务插进来,则会存放到时间格10中,currentTime再过8ms后会指向时间格10。如果同时有一个定时为19ms的任务插进来怎么办?

新来的 TimerTaskEntry会复用原来的 TimerTasklist,所以它会插入原本已经到期的时间格1。总之,整个时间轮的总体跨度是不变的,随着指针 currentTime的不断推进,当前时间轮所能处理的时间段也在不断后移,总体时间范围在 currentTime和 currentTime+ interval之间。

如果此时有一个定时为350ms的任务该如何处理?直接扩充 wheelSize的大小? Kafka中不乏几万甚至几十万毫秒的定时任务,这个 wheelsize的扩充没有底线,就算将所有的定时任务的到期时间都设定一个上限,比如100万毫秒,那么这个 wheelSize为100万毫秒的时间轮不仅占用很大的内存空间,而且也会拉低效率。 Kafka为此引入了层级时间轮的概念,当任务的到期时间超过了当前时间轮所表示的时间范围时,就会尝试添加到上层时间轮中。

如图6-8所示,复用之前的案例,第一层的时间轮 tickMs=1ms、 wheelSize=20、 interval=20ms。第二层的时间轮的 tickMs为第一层时间轮的 interval,即20ms。每一层时间轮的 wheelSize是固定的,都是20,那么第二层的时间轮的总体时间跨度 interval为400ms。以此类推,这个400ms也是第三层的 tickS的大小,第三层的时间轮的总体时间跨度为8000ms。

对于之前所说的350ms的定时任务,显然第一层时间轮不能满足条件,所以就升级到第二层时间轮中,最终被插入第二层时间轮中时间格17所对应的 TimerTaskList如果此时又有一个定时为450ms的任务,那么显然第二层时间轮也无法满足条件,所以又升级到第三层时间轮中,最终被插入第三层时间轮中时间格1的 TimerTasklist。注意到在到期时间为[400m,800ms)区间内的多个任务(比如406ms、455ms和473ms的定时任务)都会被放入第三层时间轮的时间格1，时间格1对应的 TimerTaskList的超时时间为400ms。随着时间的流逝,当此 TimerTaskList到期之时,原本定时为450ms的任务还剩下50ms的时间,还不能执行这个任务的到期操作。
这里就有一个时间轮降级的操作,会将这个剩余时间为50ms的定时任务重新提交到层级时间轮中,此时第一层时间轮的总体时间跨度不够,而第二层足够,所以该任务被放到第二层时间轮到期时间为[40ms,60ms)的时间格中。再经历40ms之后,此时这个任务又被“察觉”,不过还剩余10ms,还是不能立即执行到期操作。所以还要再有一次时间轮的降级,此任务被添加到第一层时间轮到期时间为[10m,11ms)的时间格中,之后再经历10ms后,此任务真正到期,最终执行相应的到期操作。

![](.\img\时间轮02.png)	

设计源于生活。我们常见的钟表就是一种具有三层结构的时间轮,第一层时间轮tickMs=1ms、 wheelSize=60、interval=1min,此为秒钟;第二层 tickMs=1min、wheelSize=60、interval=1hour,此为分钟;第三层 tickMs=1hour、 wheelSize=12、interval=12hours,此为时钟。

在Kafka中,第一层时间轮的参数同上面的案例一样: tickMs=1ms、wheelSize=20、interval=20ms,各个层级的 wheelSize也固定为20,所以各个层级的 tickMs和 interval也可以相应地推算出来。Kafka在具体实现时间轮 TimingWheel时还有一些小细节:

- TimingWheel在创建的时候以当前系统时间为第一层时间轮的起始时间(startMs),这里的当前系统时间并没有简单地调用System.currentTimeMillis(),而是调用了TimeSYSTEM.hiResClockMs,这是因为 currentTimeMillis()方法的时间精度依赖于操作系统的具体实现,有些操作系统下并不能达到毫秒级的精度,而Time.SYSTEM.hiResClockMs实质上采用了 System.nanoTime()/1_000_000将精度调整到毫秒级。
- TimingWheel中的每个双向环形链表 TimerTaskList都会有一个哨兵节点(sentinel),引入哨兵节点可以简化边界条件。哨兵节点也称为哑元节点(dummy node),它是个附加的链表节点,该节点作为第一个节点,它的值域中并不存储任何东西,只是为了操作的方便而引入的。如果一个链表有哨兵节点,那么线性表的第一个元素应该是链表的第二个节点。
- 除了第一层时间轮,其余高层时间轮的起始时间(startMs)都设置为创建此层时间轮时前面第一轮的 currentTime。每一层的currentTime都必须是 tickMs的整数倍,如果不满足则会将 currentTime修剪为 tickMs的整数倍,以此与时间轮中的时间格的到期时间范围对应起来。修剪方法为: currentTime= startMs-(startMs%tickMs)。 currentTime会随着时间推移而推进,但不会改变为 tickMs的整数倍的既定事实。若某一时刻的时间为 timeMs,那么此时时间轮的 currentTime= timeMs-( timeMs% tickMs),时间每推进一次,每个层级的时间轮的 currentTime都会依据此公式执行推进。
- Kafka中的定时器只需持有 TimingWheel的第一层时间轮的引用,并不会直接持有其他高层的时间轮,但每一层时间轮都会有一个引用(overflowWheel)指向更高一层的应用,以此层级调用可以实现定时器间接持有各个层级时间轮的引用。

关于时间轮的细节就描述到这里,各个组件中对时间轮的实现大同小异。读者读到这里是否会好奇文中一直描述的一个情景—“随着时间的流逝”或“随着时间的推移”,那么在Kafka中到底是怎么推进时间的呢?类似采用JDK中的 scheduleAtfixedRate来每秒推进时间轮?显然这样并不合理, TimingWheel也失去了大部分意义。

Kafka中的定时器借了JDK中的 DelayQueue来协助推进时间轮。具体做法是对于每个使用到的 TimerTaskList都加入 DelayQueue,“每个用到的 TimerTaskList”特指非哨兵节点的定时任务项 TimerTaskEntry对应的 TimerTaskList。 DelayQueue会根据 TimerTaskList对应的超时时间expiration来排序,最短 expiration的 TimerTaskList会被排在 DelayQueue的队头。Kafka中会有一个线程来获取 DelayQueue中到期的任务列表,有意思的是这个线程所对应的名称叫作“ExpiredOperationReaper”,可以直译为“过期操作收割机”。当“收割机”线程获取 DelayQueue中超时的任务列表 TimerTaskList之后,既可以根据 TimerTaskList的 expiration来推进时间轮的时间,也可以就获取的TimerTaskList执行相应的操作,对里面的 TimerTaskEntry该执行过期操作的就执行过期操作,该降级时间轮的就降级时间轮。

读到这里或许会感到困惑,开头明确指明的 DelayQueue不适合Kafka这种高性能要求的定时任务,为何这里还要引入 DelayQueue呢?注意对定时任务项 TimerEntry的插入和删除操作而言, TimingWhee时间复杂度为O(1),性能高出 DelayQueue很多,如果直接将 TimerTaskEntry插入DelayQueue,那么性能显然难以支撑。就算我们根据一定的规则将若干 TimerTaskEntry划分到 TimerList这个组中,然后将 TimerTaskList插入 DelayQueue,如果在 TimerTasklist中又要多添加一个 TimerTaskEntry时该如何处理呢?对 DelayQueue而言,这类操作显然变得力不从心。

分析到这里可以发现, Kafka中的 TimingWheel专门用来执行插入和删除 TimerTaskEntry的操作,而 DelayQueue专门负责时间推进的任务。试想一下, DelayQueue中的第一个超时任务列表的 expiration为20ms,第二个超时任务为840ms,这里获取 DelayQueue的队头只需要O(1)的时间复杂度(获取之后 DelayQueue内部才会再次切换出新的队头)。如果采用每秒定时推进,那么获取第一个超时的任务列表时执行的200次推进中有199次属于“空推进”,而获取第二个超时任务时又需要执行639次“空推进”,这样会无故空耗机器的性能资源,这里采用 DelayQueue来辅助以少量空间换时间,从而做到了“精准推进”。 Kafka中的定时器真可谓“知人善用”,用 TimingWheel做最擅长的任务添加和删除操作,而用 DelayQueue做最擅长的时间推进工作,两者相辅相成。

# 延时操作

如果在使用生产者客户端发送消息的时候将acks参数设置为-1,那么就意味着需要等待ISR集合中的所有副本都确认收到消息之后才能正确地收到响应的结果,或者捕获超时异常。

如图6-9、图6-10和图6-11所示,假设某个分区有3个副本: leader、follower和 follower2,它们都在分区的ISR集合中。为了简化说明,这里我们不考虑ISR集合伸缩的情况。 Kafka在收到客户端的生产请求(ProduceRequest)后,将消息3和消息4写入 leader副本的本地日志文件。由于客户端设置了acks为-1,那么需要等到 follower和 follower2两个副本都收到消息3和消息4后才能告知客户端正确地接收了所发送的消息。如果在一定的时间内, follower副本或 follower2副本没能够完全拉取到消息3和消息4,那么就需要返回超时异常给客户端。生产请求的超时时间由参数 request.timeout.ms配置,默认值为3000,即30s。

那么这里等待消息3和消息4写入 follower副本和 follower2副本,并返回相应的响应结果给客户端的动作是由谁来执行的呢?在将消息写入 leader副本的本地日志文件之后, Kafka会创建一个延时的生产操作( DelayedProduce),用来处理消息正常写入所有副本或超时的情况,以返回相应的响应结果给客户端。

![](.\img\延时操作01.png)	



在Kafka中有多种延时操作,比如前面提及的延时生产,还有延时拉取(DelayedFetch)、延时数据删除(DelayedDeleteRecords)等。延时操作需要延时返回响应的结果,首先它必须有个超时时间( delayS),如果在这个超时时间内没有完成既定的任务,那么就需要强制完成以返回响应结果给客户端。其次,延时操作不同于定时操作,定时操作是指在特定时间之后执行的操作,而延时操作可以在所设定的超时时间之前完成,所以延时操作能够支持外部事件的触发。就延时生产操作而言,它的外部事件是所要写入消息的某个分区的HW(高水位)发生增长。也就是说,随着 follower副本不断地与 leader副本进行消息同步,进而促使HW进一步增长,HW每増长一次都会检测是否能够完成此次延时生产操作,如果可以就执行以此返回响应结果给客户端;如果在超时时间内始终无法完成,则强制执行。

延时操作创建之后会被加入延时操作管理器(DelayedOperationPurgatory)来做专门的处理。延时操作有可能会超时,每个延时操作管理器都会配备一个定时器(SystemTimer)来做超时管理,定时器的底层就是采用时间轮( Timing Wheel)实现的。在《时间轮》中提及时间轮的轮转是靠“收割机”线程ExpiredOperationReaper来驱动的,这里的“收割机”线程就是由延时操作管理器启动的。也就是说,定时器、“收割机”线程和延时操作管理器都是一一对应的。延时操作需要支持外部事件的触发,所以还要配备一个监听池来负责监听每个分区的外部事件一一查看是否有分区的HW发生了增长。另外需要补充的是, ExpiredOperationReaper不仅可以推进时间轮,还会定期清理监听池中已完成的延时操作。

图6-12描绘了客户端在请求写入消息到收到响应结果的过程中与延时生产操作相关的细节,在了解相关的概念之后应该比较容易理解:如果客户端设置的acκs参数不为-1,或者没有成功的消息写入,那么就直接返回结果给客户端,否则就需要创建延时生产操作并存入延时操作管理器,最终要么由外部事件触发,要么由超时触发而执行。

![](.\img\延时操作02.png)	

有延时生产就有延时拉取。以图6-13为例,两个 follower副本都已经拉取到了 leader副本的最新位置,此时又向 leader副本发送拉取请求,而 leader副本并没有新的消息写入,那么此时 leader副本该如何处理呢?可以直接返回空的拉取结果给 follower副本,不过在 leader副本直没有新消息写入的情况下, follower副本会一直发送拉取请求,并且总收到空的拉取结果,这样徒耗资源,显然不太合理。

![](.\img\延时操作03.png)	

Kafka选择了延时操作来处理这种情况。 Kafka在处理拉取请求时,会先读取一次日志文件,如果收集不到足够多( fetchMinBytes,由参数 fetch.min.bytes配置,默认值为1)的消息,那么就会创建一个延时拉取操作(DelayedFetch)以等待拉取到足够数量的消息。当延时拉取操作执行时,会再读取一次日志文件,然后将拉取结果返回给 follower副本。延时拉取操作也会有一个专门的延时操作管理器负责管理,大体的脉络与延时生产操作相同,不再赘述。如果拉取进度一直没有追赶上 leader副本,那么在拉取 leader副本的消息时一般拉取的消息大小都会不小于 fetchMinBytes,这样Kafka也就不会创建相应的延时拉取操作,而是立即返回拉取结果。

延时拉取操作同样是由超时触发或外部事件触发而被执行的。超时触发很好理解,就是等到超时时间之后触发第二次读取日志文件的操作。外部事件触发就稍复杂了一些,因为拉取请求不单单由 follower副本发起,也可以由消费者客户端发起,两种情况所对应的外部事件也是不同的。如果是 follower副本的延时拉取,它的外部事件就是消息追加到了 leader副本的本地日志文件中;如果是消费者客户端的延时拉取,它的外部事件可以简单地理解为HW的增长,目前版本的Kafka还引入了事务的概念,对于消费者或 follower副本而言,其默认的事务隔离级别为“read_uncommitted”。不过消费者可以通过客户端参数 isolation.leve将事务隔离级别设置为“read_committed”(注意: follower副本不可以将事务隔离级别修改为这个值),这样消费者拉取不到生产者已经写入却尚未提交的消息。对应的消费者的延时拉取,它的外部事件实际上会切换为由LSO(LastStableOffset的增长来触发。LSO是HW之前除去未提交的事务消息的最大偏移量,LSO≤HW,有关事务和LSO的内容可以分别参考《事务》和《Kafka监控》节。

# 控制器

在 Kafka集群中会有一个或多个 broker,其中有一个 broker会被选举为控制器(KafkaController),它负责管理整个集群中所有分区和副本的状态。当某个分区的 leader副本出现故障时,由控制器负责为该分区选举新的 leader副本。当检测到某个分区的ISR集合发生变化时,由控制器负责通知所有 broker更新其元数据信息。当使用 kafka-topics.sh脚本为某个 topic增加分区数量时,同样还是由控制器负责分区的重新分配。

## 控制器的选举及异常恢复

Kafka中的控制器选举工作依赖于 ZooKeeper,成功竞选为控制器的 broker会在 ZooKeeper中创建/controller这个临时(EPHEMERAL)节点,此临时节点的内容参考如下:
{"version": 1,"brokerid":0,"timestamp":"1529210278988"}
其中 version在目前版本中固定为1, brokerid表示成为控制器的 broker的id编号timestamp表示竞选成为控制器时的时间戳。

在任意时刻,集群中有且仅有一个控制器。每个 broker启动的时候会去尝试读取/controller节点的 brokerid的值,如果读取到 brokerid的值不为-1,则表示已经有其他 broker节点成功竞选为控制器,所以当前 broker就会放弃竞选;如果 ZooKeeper中不存在/controller节点,或者这个节点中的数据异常,那么就会尝试去创建/controller节点。当前 broker去创建节点的时候,也有可能其他 broker同时去尝试创建这个节点,只有创建成功的那个 broker才会成为控制器,而创建失败的 broker竞选失败。每个 broker都会在内存中保存当前控制器的 brokerId值,这个值可以标识为 activeControllerld。

ZooKeeper中还有一个与控制器有关的/controller_epoch节点,这个节点是持久(PERSISTENT)节点,节点中存放的是一个整型的 controller_epoch值。 controller_epoch用于记录控制器发生变更的次数,即记录当前的控制器是第几代控制器,我们也可以称之为“控制器的纪元”。

controller_epoch的初始值为1,即集群中第一个控制器的纪元为1,当控制器发生变更时,每选出一个新的控制器就将该字段值加1。每个和控制器交互的请求都会携带 controller_epoch这个字段,如果请求的 controller_epoch值小于内存中的 controller_epoch值,则认为这个请求是向已经过期的控制器所发送的请求,那么这个请求会被认定为无效的请求。如果请求的 controller_epoch值大于内存中的 controller_epoch值,那么说明已经有新的控制器当选了。由此可见,Kafka通过 controller_epoch来保证控制器的唯一性,进而保证相关操作的一致性。

具备控制器身份的 broker需要比其他普通的 broker多一份职责,具体细节如下:

- 监听分区相关的变化。为 ZooKeeper中的/admin/reassign_partitions节点注册 PartitionReassignmentHandler,用来处理分区重分配的动作。为 ZooKeeper中的/isr_change_notification节点注册 IsrChangeNotificetionHandler,用来处理ISR集合变更的动作。为 ZooKeeper中的/admin/preferred-replica-election节点添加 PreferredReplicaElectionHandler,用来处理优先副本的选举动作。
- 监听主题相关的变化。为 ZooKeeper中的/brokers/topics节点添加Topic ChangeHandler,用来处理主题增减的变化;为 ZooKeeper中的/admin/delete_topics节点添加 TopicDeletionHandler,用来处理删除主题的动作。
- 监听 broker相关的变化。为ZooKeeper中的/brokers/ids节点添加 BrokerChangeHandler,用来处理 broker增减的变化。
- 从 ZooKeeper中读取获取当前所有与主题、分区及 broker有关的信息并进行相应的管理。对所有主题对应的 ZooKeeper中的/brokers/topics/<topic>节点添加PartitionModificationsHandler,用来监听主题中的分区分配变化。
- 启动并管理分区状态机和副本状态机。
- 更新集群的元数据信息。
- 如果参数auto.leader.rebalance.enable设置为true,则还会开启一个名为“auto-leader-rebalance-task”的定时任务来负责维护分区的优先副本的均衡。

控制器在选举成功之后会读取 ZooKeeper中各个节点的数据来初始化上下文信息(ControllerContext),并且需要管理这些上下文信息。比如为某个主题增加了若干分区,控制器在负责创建这些分区的同时要更新上下文信息,并且需要将这些变更信息同步到其他普通的broker节点中。不管是监听器触发的事件,还是定时任务触发的事件,或者是其他事件(比如ControlledShutdown,具体可以参考《优雅关闭》一节)都会读取或更新控制器中的上下文信息,那么这样就会涉及多线程间的同步。如果单纯使用锁机制来实现,那么整体的性能会大打折扣。针对这一现象, Kafka的控制器使用单线程基于事件队列的模型,将每个事件都做一层封装,然后按照事件发生的先后顺序暂存到 LinkedBlockingQueue中,最后使用一个专用的线程(ControllerEventThread)按照FIFO(First Input First Output,先入先出)的原则顺序处理各个事件,这样不需要锁机制就可以在多线程间维护线程安全,具体可以参考图6-14。

在Kafka的早期版本中,并没有采用 KafkaController这样一个概念来对分区和副本的状态进行管理,而是依赖于 ZooKeeper,每个 broker都会在 ZooKeeper上为分区和副本注册大量的监听器(Watcher)。当分区或副本状态变化时,会唤醒很多不必要的监听器,这种严重依赖ZooKeeper的设计会有脑裂、羊群效应,以及造成 Zookeeper过载的隐患(旧版的消费者客户端存在同样的问题,详细内容参考7.2.1节)。在目前的新版本的设计中,只有 KafkaController在 ZooKeeper上注册相应的监听器,其他的 broker极少需要再监听 ZooKeeper中的数据变化,这样省去了很多不必要的麻烦。不过每个broker还是会对/controller节点添加监听器,以此来监听此节点的数据变化(ControllerChangeHandler)。

![](.\img\控制器01.png)	

当/controller节点的数据发生变化时,每个 broker都会更新自身内存中保存的activeControllerld。如果 broker在数据变更前是控制器,在数据变更后自身的 brokerid值与新的 activeControllerId值不一致,那么就需要“退位”,关闭相应的资源,比如关闭状态机、注销相应的监听器等。有可能控制器由于异常而下线,造成/controller这个临时节点被自动删除;也有可能是其他原因将此节点删除了。

当/controller节点被删除时,每个 broker都会进行选举,如果 broker在节点被删除前是控制器,那么在选举前还需要有一个“退位”的动作。如果有特殊需要,则可以手动删除/controller节点来触发新一轮的选举。当然关闭控制器所对应的 broker,以及手动向/controller节点写入新的 brokerid的所对应的数据,同样可以触发新一轮的选举。

## 优雅关闭

如何优雅地关闭Kafka?笔者在做测试的时候经常性使用jps(或者ps ax)配合kill -9的方式来快速关闭 Kafka broker的服务进程,显然kill -9这种“强杀”的方式并不够优雅,它并不会等待 Kafka进程合理关闭一些资源及保存一些运行数据之后再实施关闭动作。在有些
场景中,用户希望主动关闭正常运行的服务,比如更换硬件、操作系统升级、修改 Kafka配置等。如果依然使用上述方式关闭就略显粗暴。

那么合理的操作应该是什么呢?Kafka自身提供了一个脚本工具,就是存放在其bin目录下的 kafka-server-stop.sh,这个脚本的内容非常简单,具体内容如下:
PIDS=$(ps ax I grep -i 'kafka\.Kafka' | grep java | grep -v grep | awk '{print $1}')
if [ -z "$PIDS"]; then
  echo "No kafka server to stop "
  exit 1
else
  kill -s TERM $PIDS
fi

可以看出 kafka-server-stop.sh首先通过ps ax的方式找出正在运行Kafka的进程号PIDS,然后使用kill -s TERM $PIDS的方式来关闭。但是这个脚本在很多时候并不奏效,这一点与ps命令有关系。在 Linux操作系统中,ps命令限制输出的字符数不得超过页大小 PAGE_SIZE,一般CPU的内存管理单元(Memory Management Unit,简称MMU)的PAGE_SIZE为4096。也就是说,ps命令的输出的字符串长度限制在4096内,这会有什么问题呢?

细心的读者可以留意到白色部分中的信息并没有打印全,因为已经达到了4096的字符数的限制。而且打印的信息里面也没有 kafka-server-stop.sh中 ps ax | grep -i 'kafka\.Kafka'所需要的“kafka.Kafka”这个关键字段,因为这个关键字段在4096个字符的范围之外。与Kafka进程有关的输出信息太长,所以 kafka-server-stop.sh脚本在很多情况下并不会奏效。

注意要点: Kafka服务启动的入口就是 kafka.Kafka,采用 Scala语言编写 object。

那么怎么解决这种问题呢?

这里我们可以直接修改 kafka-server-stop.sh脚本的内容,将其中的第一行命令修改如下:
PIDS=$(ps ax | grep -i 'kafka' | grep java | grep -v grep | awk '{print $1}')

即把“\.Kafka”去掉,这样在绝大多数情况下是可以奏效的。如果有极端情况,即使这样也不能关闭,那么只需要按照以下两个步骤就可以优雅地关闭Kafka的服务进程:
(1)获取Kafka的服务进程号PIDS。可以使用Java中的jps命令或使用 Linux系统中的ps命令来查看。
(2)使用kill -s TERM $PIDS或kill -15 $PIDS的方式来关闭进程,注意千万不要使用kill -9的方式。

为什么这样关闭的方式会是优雅的? Kafka服务入口程序中有一个名为“kafka-shutdown-hock”的关闭钩子,待 Kafka进程捕获终止信号的时候会执行这个关闭钩子中的内容,其中除了正常关闭一些必要的资源,还会执行一个控制关闭(ControlledShutdown)的动作。使用ControlledShutdown的方式关闭 Kafka有两个优点:一是可以让消息完全同步到磁盘上,在服务下次重新上线时不需要进行日志的恢复操作;二是 ControllerShutdown在关闭服务之前,会对其上的 leader副本进行迁移,这样就可以减少分区的不可用时间。

若要成功执行 ControlledShutdown动作还需要有一个先决条件,就是参数 controlled.shutdown.enable的值需要设置为true,不过这个参数的默认值就为true,即默认开始此项功能。ControlledShutdown动作如果执行不成功还会重试执行,这个重试的动作由参数
controlled.shutdown.max.retries配置,默认为3次,每次重试的间隔由参数controlled.shutdown.retry.backoff.ms设置,默认为5000ms。

下面我们具体探讨 ControlledShutdown的整个执行过程。

参考图6-16,假设此时有两个 broker,其中待关闭的 broker的id为x, Kafka控制器所对应的 broker的id为y。待关闭的 broker在执行 ControlledShutdown动作时首先与Kafka控制器建立专用连接(对应图6-16中的步骤①),然后发送 ControlledShutdownRequest请求,ControlledShutdownRequest请求中只有一个 brokerid字段,这个 brokerid字段的值设置为自身的 brokerid的值,即x(对应图6-16中的步骤②)。

Kafka控制器在收到 ControlledShutdownRequest请求之后会将与待关闭 broker有关联的所有分区进行专门的处理,这里的“有关联”是指分区中有副本位于这个待关闭的 broker之上(这里会涉及Kafka控制器与待关闭 broker之间的多次交互动作,涉及 leader副本的迁移和副本的关闭动作,对应图6-16中的步骤③)。

![](.\img\控制器02.png)	

ControlledShutdownRequest的结构如图6-17所示。

![](.\img\控制器03.png)	

如果这些分区的副本数大于1且 leader副本位于待关闭 broker上,那么需要实施 leader副本的迁移及新的SR的变更。具体的选举分配的方案由专用的选举器 ControlledShutdownLeaderSelector提供,有关选举的细节可以参考延时操作的内容。

如果这些分区的副本数只是大于1, leader副本并不位于待关闭 broker上,那么就由Kafka控制器来指导这些副本的关闭。如果这些分区的副本数只是为1,那么这个副本的关闭动作会在整个 ControlledShutdown动作执行之后由副本管理器来具体实施。

对于分区的副本数大于1且 leader副本位于待关闭 broker上的这种情况,如果在Kafka控制器处理之后 leader副本还没有成功迁移,那么会将这些没有成功迁移 leader副本的分区记录下来,并且写入 ControlledShutdownResponse的响应(对应图6-16中的步骤④,整个ControlledShutdown动作是一个同步阻塞的过程)。 ControlledShutdownResponse的结构如图6-18所示。

![](.\img\控制器04.png)	

待关闭的 broker在收到 ControlledShutdownResponse响应之后,需要判断整个 ControlledShutdown动作是否执行成功,以此来进行可能的重试或继续执行接下来的关闭资源的动作。执行成功的标准是 ControlledShutdownResponse中 error_code字段值为0,并且 partitions_remaining数组字段为空。

注意要点:图6-16中也有可能ⅹ=y,即待关闭的 broker同时是Kafka控制器,这也就意味着自己可以给自己发送 ControlledShutdownRequest请求,以及等待自身的处理并接收ControlledShutdownResponse的响应,具体的执行细节和x!=y的场景相同。

在了解了整个 ControlledShutdown动作的具体细节之后,我们不难看出这一切实质上都是由 ControlledShutdownRequest请求引发的,我们完全可以自己开发一个程序来连接Kafka控制器,以此来模拟对某个 broker实施 ControlledShutdown的动作。为了实现方便,我们可以对KafkaAdminClient做一些扩展来达到目的。

首先参考 org.apache.kafka.clients.admin.AdminClient接口中的惯有编码样式来添加两个方法:

![](.\img\控制器05.png)	

第一个方法中的 ControlledShutdownOptions和 ControlledShutdownResult都是 KafkaAdminClient的惯有编码样式, ControlledShutdownOptions中没有实质性的内容,具体参考如下

![](.\img\控制器06.png)	

ControlledShutdownResult中没有像 KafkaAdminClient中惯有的那样对 ControlledShutdownResponse进行细致化的处理,而是直接将 ControlledShutdownResponse暴露给用户,这样用户可以更加细腻地操控内部的细节。

第二个方法中的参数Node是我们需要执行 ControlledShutdown动作的 broker节点,Node的构造方法至少需要三个参数:id、host和port,分别代表所对应的 broker的id编号、IP地址和端口号。一般情况下,对用户而言,并不一定清楚这个三个参数的具体值,有的要么只知道要关闭的 broker的IP地址和端口号,要么只清楚具体的id编号,为了程序的通用性,我们还需要做进一步的处理。详细看一下 org.apache.kafka.clients.admin.KafkaAdminClient中的具体做法：

![](.\img\控制器07.png)	

我们可以看到在内部的 createRequest方法中对Node的id做了一些处理,因为对ControlledShutdownRequest协议的包装只需要这个id的值。程序中首先判断Node的id是否大于0,如果不是则需要根据host和port去 KafkaAdminClient缓存的元数据 metadata中查
找匹配的id。注意到代码里还有一个标粗的 ControllerNodeProvider,它提供了 Kafka控制器对应的节点信息,这样用户只需要提供Kafka集群中的任意节点的连接信息,不需要知晓具体的Kafka控制器是谁。

最后我们再用一段测试程序来模拟发送 ControlledShutdownRequest请求及处理ControlledShutdownResponse,详细参考如下:

![](.\img\控制器08.png)	

其中 brokerUrl是连接的任意节点,node是需要关闭的 broker节点,当然这两个可以是同一个节点,即代码中的 hostname1等于 hostname2。使用 KafkaAdminClient的整个流程为:首先连接集群中的任意节点;接着通过这个连接向 Kafka集群发起元数据请求
(MetadataRequest)来获取集群的元数据 metadata;然后获取需要关闭的 broker节点的id,如果没有指定则去 metadata中查找,根据这个jd封装 ControlledShutdownRequest请求;之后再去metadata中査找 Kafka控制器的节点,向这个 Kafka控制器节点发送请求;最后等待 Kafka控制器的 ControlledShutdownResponse响应并做相应的处理。

注意 ControlledShutdown只是关闭 Kafka broker的一个中间过程,所以不能寄希望于只使用 ControlledShutdownRequest请求就可以关闭整个 Kafka broker的服务进程。



## 分区Leader的选举

分区 leader副本的选举由控制器负责具体实施。当创建分区(创建主题或增加分区都有创建分区的动作)或分区上线(比如分区中原先的 leader副本下线,此时分区需要选举一个新的leader上线来对外提供服务)的时候都需要执行 leader的选举动作,对应的选举策略为OfflinePartitionLeaderElectionStrategy。这种策略的基本思路是按照AR集合中副本的顺序査找第一个存活的副本,并且这个副本在ISR集合中。一个分区的AR集合在分配的时候就被指定,并且只要不发生重分配的情况,集合内部副本的顺序是保持不变的,而分区的ISR集合中副本的顺序可能会改变。

注意这里是根据AR的顺序而不是ISR的顺序进行选举的。举个例子,集群中有3个节点:
broker0、 broker1和 broker2,在某一时刻具有3个分区且副本因子为3的主题 topic-leader的具体信息如下:

![](.\img\控制器09.png)	

此时关闭 broker0,那么对于分区2而言,存活的AR就变为[1,2],同时IsR变为[2,1]。此时查看主题 topic-leader的具体信息(参考如下),分区2的 leader就变为了1而不是2。

![](.\img\控制器10.png)	

如果ISR集合中没有可用的副本,那么此时还要再检查一下所配置的 unclean.leader.election.enable参数(默认值为 false)。如果这个参数配置为tue,那么表示允许从非ISR列表中的选举 leader,从AR列表中找到第一个存活的副本即为 leader当分区进行重分配(可以先回顾一下4.32节的内容)的时候也需要执行 leader的选举动作,对应的选举策略为 ReassignPartitionLeaderElectionStrategy。这个选举策略的思路比较简单:从重分配的AR列表中找到第一个存活的副本,且这个副本在目前的ISR列表中。

当发生优先副本(可以先回顾一下《Kafka分区管理》内容)的选举时,直接将优先副本设置为 leader即可,AR集合中的第一个副本即为优先副本(PreferredReplicaPartitionLeaderElectionStrategy)。

还有一种情况会发生 leader的选举,当某节点被优雅地关闭(也就是执行ControlledShutdown)时,位于这个节点上的 leader副本都会下线,所以与此对应的分区需要执行 leader的选举。与此对应的选举策略(ControlledShutdownPartitionStrategy)为:从AR列表中找到第一个存活的副本,且这个副本在目前的ISR列表中,与此同时还要确保这个副本不处于正在被关闭的节点上。

# 参数解密

如果 broker端没有显式配置listeners(或 advertised.listeners)使用IP地址,那么最好将 bootstrap.server配置成主机名而不要使用IP地址,因为 Kafka内部使用的是全称域名(Fully Qualified Domain Name)。如果不统一,则会出现无法获取元数据的异常。

## broker.id

broker.id是 broker在启动之前必须设定的参数之一,在 Kafka集群中,每个 broker都有唯一的id(也可以记作 brokerId)值用来区分彼此。 broker在启动时会在 ZooKeeper中的/brokers/ids路径下创建一个以当前 brokerId为名称的虚节点, broker的健康状态检查就依赖于此虚节点。当 broker下线时,该虚节点会自动删除,其他 broker节点或客户端通过判断/brokers/ids路径下是否有此 broker的 brokerId节点来确定该 broker的健康状态。

可以通过 broker端的配置文件 config/server.properties里的 broker.id参数来配置brokerId,默认情况下 broker.id值为-1。在 Kafka中, brokerId值必须大于等于0才有可能正常启动,但这里并不是只能通过配置文件 config/server.properties来设定这个值,还可以通过meta.properties文件或自动生成功能来实现。

首先了解一下 meta.properties文件, meta.properties文件中的内容参考如下:
\# Sun May2723:03:04csT2018
version=0
broker.id=0

meta.properties件中记录了与当前Kaa版本对应的一个 version字段,不过目前只有一个为0的固定值。还有一个 broker.id,即brokerId值。 broker在成功启动之后在每个日志根目录下都会有一个 meta.properties文件。meta.properties文件与 broker.id的关联如下:
(1)如果log.dir或log.dirs中配置了多个日志根目录,这些日志根目录中的meta.properties文件所配置的 broker.id不一致则会抛出 InconsistentBrokerIdException的异常。
(2)如果 config/server.properties配置文件里配置的 broker.id的值和 meta.properties文件里的 broker.id值不一致,那么同样会抛出 InconsistentBrokerldException的异常。
(3)如果 config/server.properties配置文件中并未配置 broker.id的值,那么就以meta.properties文件中的 broker.id值为准。
(4)如果没有 meta.properties文件,那么在获取合适的 broker.id值之后会创建一个新的 meta.properties文件并将 broker.id值存入其中。

如果 config/server.properties配置文件中并未配置 broker.id,并且日志根目录中也没有任何 meta.properties文件(比如第一次启动时),那么应该如何处理呢?

Kafka还提供了另外两个 broker端参数: broker.id.generation.enable和reserved.broker.max.id来配合生成新的 brokerld。broker.id.generation.enable参数用来配置是否开启自动生成 brokerId的功能,默认情况下为true,即开启此功能。自动生成的 brokerId有一个基准值,即自动生成的 brokerId必须超过这个基准值,这个基准值通过reserverd.broker.max.id参数配置,默认值为1000。也就是说,默认情况下自动生成的brokerId从1001开始。

自动生成的 brokerId的原理是先往 ZooKeeper中的/brokers/said节点中写入一个空字符串,然后获取返回的Stat信息中的 version值,进而将 version的值和reserved.broker.max.id参数配置的值相加。先往节点中写入数据再获取Stat信息,这样可以确保返回的 version值大于0,进而就可以确保生成的 brokerId值大于reserved.broker.max.id参数配置的值,符合非自动生成的 broker.id的值在[0,reserved.broker.max.id区间设定。

初始化时 ZooKeeper中/brokers/seqid节点的状态如下:

![](.\img\参数解密01.png)	

可以看到 dataVersion=0,这个就是前面所说的 version。在插入一个空字符串之后,dataVersion就自增1,表示数据发生了变更,这样通过 ZooKeeper的这个功能来实现集群层面的序号递增,整体上相当于一个发号器。

![](.\img\参数解密02.png)	

大多数情况下我们一般通过并且习惯于用最普通的 config/server.properties配置文件的方式来设定 brokerId的值,如果知晓其中的细枝末节,那么在遇到诸如 InconsistentBrokerldException异常时就可以处理得游刃有余,也可以通过自动生成 brokerId的功能来实现一些另类的功能。

## bootstrap.servers

bootstrap.servers不仅是 Kafka producer、 Kafka Consumer客户端中的必备参数,而且在 Kafka Connect、 Kafka streams和 KafkaAdminClient中都有涉及,是一个至关重要的参数。如果你使用过旧版的生产者或旧版的消费者客户端,那么你可能还会对 bootstrap.servers相关的另外两个参数 metada.broker.list和 zookeeper.connect有些许印象,这3个参数也见证了Kafka的升级变迁。

我们一般可以简单地认为 bootstrap.servers这个参数所要指定的就是将要连接的Kafka集群的 broker地址列表。不过从深层次的意义上来讲,这个参数配置的是用来发现 Kafka集群元数据信息的服务地址。为了更加形象地说明问题,我们先来看一下图6-19。

![](.\img\参数解密03.png)	

客户端 KafkaProducer1与 Kafka Cluster直连,这是客户端给我们的既定印象,而事实上客户端连接 Kafka集群要经历以下3个过程,如图6-19中的右边所示。
(1)客户端 KafkaProducer2与 bootstrap.servers参数所指定的 Server连接,并发送MetadataRequest请求来获取集群的元数据信息
(2)Server在收到 MetadataRequest请求之后,返回 MetadataResponse给 KafkaProducer2,在 MetadataResponse中包含了集群的元数据信息。
(3)客户端 KafkaProducer2收到的 Metadata Response之后解析出其中包含的集群元数据信息,然后与集群中的各个节点建立连接,之后就可以发送消息了。

在绝大多数情况下, Kafka本身就扮演着第一步和第二步中的 Server角色,我们完全可以将这个 Server的角色从Kafka中剥离出来。我们可以在这个 Server的角色上大做文章,比如添加一些路由的功能、负载均衡的功能。

下面演示如何将 Server的角色与 Kafka分开。默认情况下,客户端从 Kafka中的某个节点来拉取集群的元数据信息,我们可以将所拉取的元数据信息复制一份存放到 Server中,然后对外提供这份副本的内容信息。

由此可见,我们首先需要做的就是获取集群信息的副本,可以在Kafa的 org.apache.kafka.commonrequest.MetadataResponse的构造函数中嵌入代码来复制信息, MetadataResponse的构造函数如下所示。

![](.\img\参数解密04.png)	

获取集群元数据的副本之后,我们就可以实现一个服务程序来接收 MetadataRequest请求和返回 MetadataResponse,从零开始构建一个这样的服务程序也需要不少的工作量,需要实现对MetadataRequest与 MetadataResponse相关协议解析和包装,这里不妨再修改一下Kafka的代码,让其只提供 Server相关的内容。整个示例的架构如图6-20所示。

![](.\img\参数解密05.png)	

为了演示方便,图6-20中的 Kafka Cluster1和 Kafka Cluster2都只包含一个 broker节点。Kafka Cluster1扮演的是 Server的角色,下面我们修改它的代码让其返回 Kafka Cluster2的集群元数据信息。假设我们已经通过前面一步的操作获取了 Kafka Cluster2的集群元数据信息,在Kafka ClusterI中将这份副本回放。

在Kafka的代码 kafka.server.KafkaApis中有关专门处理元数据信息的方法如下所示。

def handleTopicMetadataRequest(request: Requestchannel.Request)
我们将这个方法内部的最后一段代码替换,详情如下：

![](.\img\参数解密06.png)	

上面示例代码中有“//”注释的是原本的代码实现,没有“//”注释的两行代码是我们修改后的代码实现,代码里的BootstrapServerParam. getMetadata()方法也是需要自定义实现的,这个方法返回的就是从 Kafka Cluster2中获取的元数据信息的副本回放,BootstrapServerParam的实现如下：

![](.\img\参数解密07.png)	

示例代码中用了最笨的方法来创建了一个 MetadataResponse,如果我们在复制 Kafka Cluster2元数据信息的时候使用了某种序列化手段,那么在这里我们就简单地执行一下反序列化来创建一个 MetadataResponse对象。

修改完 Kafka Cluster1的代码之后我们将它和 Kafka Cluster2都启动起来,然后创建一个生产者 KafkaProducer来持续发送消息,这个 Kafkaproducer中的 bootstrap.servers参数配置为 Kafka Cluster的服务地址。我们再创建一个消费者 Kafka Consumer来持续消费消息,这个 KafkaConsumer中的 bootstrap.servers参数配置为 Kafka cluster2的服务地址。

实验证明, KafkaProducer中发送的消息都流入 Kafka Cluster2并被 KafkaConsumer消费。查看 Kafka Cluster1中的日志文件,发现并没有消息流入。如果此时我们再关闭 Kafka Cluster1的服务,会发现 KafkaProducer和 KafkaConsumer都运行完好,已经完全没有 Kafka Cluster1的任何事情了。

这里只是为了讲解 bootstrap.servers参数所代表的真正含义而做的一些示例演示,笔者并不建议在真实应用中像示例中的一样分离出 Server的角色。

在旧版的生产者客户端(Scala版本)中还没有 bootstrap.servers这个参数,与此对应的是 metadata.broker.list参数。 metadata.broker.list这个参数很直观, metadata表示元数据, broker.list表示 broker的地址列表,从取名我们可以看出这个参数很直接地表示所要连接的 Kafka Broker的地址,以此获取元数据。而新版的生产者客户端中的bootstrap.servers参数的取名显然更有内涵,可以直观地翻译为“引导程序的服务地址”,这样在取名上就多了一层“代理”的空间,让人可以遐想出 Server角色与Kafka分离的可能。

在旧版的消费者客户端(Scala版本)中也没有 bootstrap.servers这个参数,与此对应的是 zookeeper.connect参数,意为通过ZooKeeper来建立消费连接。

很多读者从0.8.x版本开始沿用到现在的2.0.0版本,对于版本变迁的客户端中出现的bootstrap.servers、metadata.broker.list、zookeeper.connect参数往往不是很清楚。这一现象还存在 Kafka所提供的诸多脚本之中,在这些脚本中连接 Kafka采用的选项
参数有--bootstrap-server、--broker-list和--zokeeper(分别与前面的3个参数对应),这让很多Kafka的老手也很难分辨哪个脚本该用哪个选项参数。

--bootstrap-server是一个逐渐盛行的选项参数,这一点毋庸置疑。而--broker-list已经被淘汰,但在2.0.0版本中还没有完全被摒弃,在kafka-console-producer.sh脚本中还是使用的这个选项参数,在后续的 Kafka版本中可能会被替代为--bootstrap-server。
--zookeeper这个选项参数也逐渐被替代,在目前的2.0.0版本中, kafka-console-consumer. sh中已经完全没有了它的影子,但并不意味着这个参数在其他脚本中也被摒弃了。在kafka-topics.sh脚本中还是使用的--zookeeper这个选项参数,并且在未来的可期版本中也不见得会被替换,因为 kafka-topics.sh脚本实际上操纵的就是 ZooKeeper中的节点,而不是Kafka本身,它并没有被替代的必要。

## 服务端参数列表

表6-6列出了部分服务端重要参数。

![](.\img\参数解密08.png)	





