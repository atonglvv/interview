# Nacos作为服务注册中心如何避免并发读写冲突问题？

Nacos 注册表通过分布式锁机制来防止多节点读写并发冲突，保证数据的一致性和可靠性。

Nacos 在注册表服务的实现中，使用了基于数据库的乐观锁和基于 Zookeeper 的悲观锁两种不同的锁机制。

# Nacos是AP还是CP？

Nacos默认是AP，支持AP和CP模式的切换。

如果注册Nacos的client节点注册时ephemeral=true，那么Nacos集群对这个client节点的效果就是AP，采用distro协议实现；而注册Nacos的client节点注册时ephemeral=false，那么Nacos集群对这个节点的效果就是CP的，采用raft协议实现。根据client注册时的属性，AP，CP同时混合存在，只是对不同的client节点效果不同。Nacos可以很好的解决不同场景的业务需求。

