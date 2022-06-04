# zookeeper是什么？

ZooKeeper 是一个开源的分布式协调服务。

Zookeeper最早起源于雅虎研究院的一个研究小组。

ZooKeeper 是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。

# zookeeper可以做什么？

注册中心。（最常用）

Hadoop生态成员之一。

HBase使用ZooKeeper跟踪分布式数据的状态。

分布式锁。

# zookeeper的数据结构

Zookeeper将所有数据存储在内存中，数据模型是一棵树（Znode Tree)。

# 几个重要概念

## Session

Session 指的是 ZooKeeper  服务器与客户端会话。**在 ZooKeeper 中，一个客户端连接是指客户端和服务器之间的一个 TCP 长连接**。

**通过这个连接，客户端能够通过心跳检测与服务器保持有效的会话，也能够向Zookeeper服务器发送请求并接受响应，同时还能够通过该连接接收来自服务器的Watch事件通知。** 

Session的`sessionTimeout`值用来设置一个客户端会话的超时时间。当由于服务器压力太大、网络故障或是客户端主动断开连接等各种原因导致客户端连接断开时，**只要在`sessionTimeout`规定的时间内能够重新连接上集群中任意一台服务器，那么之前创建的会话仍然有效。**

在为客户端创建会话之前，服务端首先会为每个客户端都分配一个sessionID。由于 sessionID 是 Zookeeper 会话的一个重要标识，许多与会话相关的运行机制都是基于这个 sessionID 的，因此，无论是哪台服务器为客户端分配的 sessionID，都务必保证全局唯一。

## Znode

将数据模型中的数据单元，我们称之为数据节点——ZNode。

在Zookeeper中，node可以分为持久节点和临时节点两类。

临时节点的生命周期和客户端会话绑定，一旦客户端会话失效，那么这个客户端创建的所有临时节点都会被移除。

ZooKeeper还允许用户为每个节点添加一个特殊的属性：**SEQUENTIAL**。一旦节点被标记上这个属性，那么在这个节点被创建的时候，Zookeeper会自动在其节点名后面追加上一个整型数字，这个整型数字是一个由父节点维护的自增数字。

## Version

对应于每个ZNode，Zookeeper 都会为其维护一个叫作 **Stat** 的数据结构，Stat中记录了这个 ZNode 的三个数据版本，分别是version（当前ZNode的版本）、cversion（当前ZNode子节点的版本）和 cversion（当前ZNode的ACL版本）。

## Watcher



# 为什么最好使用奇数台服务器构成 ZooKeeper 集群？

在Zookeeper中 Leader 选举算法采用了Zab协议。Zab核心思想是当超过半数Server 写成功，则任务数据写成功。

考虑场景：

如果有3个Server，则最多允许1个Server 挂掉。

如果有4个Server，则同样最多允许1个Server挂掉。

既然3个或者4个Server，同样最多允许1个Server挂掉，为何不选择3个Server？

