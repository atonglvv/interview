# MVCC

Multi-Version Concurrency Control。

MVCC 的核心思路就是保存数据行的历史版本，通过对数据行的多个版本进行管理来实现数据库的并发控制。

# 版本链

对于Innodb来说，它的聚簇索引记录中都包含下面两个必要的隐藏列（row_id 并不是必要的）：

- trx_id：一个事务每次对某条聚簇索引记录进行改动时，都会把事务的事务id赋值给trx_id隐藏列。
- roll_pointer：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到undo日志中。roll_pointer就相当于一个指针，可以通过它找到记录修改前的信息。



