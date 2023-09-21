# Kafka先持久化还是先同步副本？

# Kafka中Zookeeper的作用？

kafka集群中有一个broker会被选举成Controller，负责管理集群broker的上下线、所有topic的分区副本分配和leader选举。

Controller管理依赖于Zookeeper。

# topic 与 partition

Topic-Partition为一对多。

topic 是逻辑上的概念，而partition 是物理上的概念，每个partition 对应一个log 文件，该log 文件中存储的就是producer 生产的数据。Producer 生产的数据会被不断追加到该log 文件末端，且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费。