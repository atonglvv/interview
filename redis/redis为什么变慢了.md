# Redis为什么变慢了？如何排查Redis性能问题

你也许或多或少地，也遇到过以下这些场景：

- 在 Redis 上执行同样的命令，为什么有时响应很快，有时却很慢？
- 为什么 Redis 执行 SET、DEL 命令耗时也很久？
- 为什么我的 Redis 突然慢了一波，之后又恢复正常了？
- 为什么我的 Redis 稳定运行了很久，突然从某个时间点开始变慢了？

## Redis真的变慢了吗？

从你的业务服务到 Redis 这条链路变慢的原因可能也有 2 个：

1. 业务服务器到 Redis 服务器之间的网络存在问题，例如网络线路质量不佳，网络数据包在传输时存在延迟、丢包等情况
2. Redis 本身存在问题，需要进一步排查是什么原因导致 Redis 变慢

通常来说，第一种情况发生的概率比较小，如果是服务器之间网络存在问题，那部署在这台业务服务器上的所有服务都会发生网络延迟的情况，此时你需要联系网络运维同事，让其协助解决网络问题。

我们这篇文章，重点关注的是第二种情况。

排除网络原因，如何确认你的 Redis 是否真的变慢了？

首先，你需要对 Redis 进行基准性能测试，了解你的 Redis 在生产环境服务器上的基准性能。

什么是基准性能？

简单来讲，基准性能就是指 Redis 在一台负载正常的机器上，其最大的响应延迟和平均响应延迟分别是怎样的？

为什么要测试基准性能？我参考别人提供的响应延迟，判断自己的 Redis 是否变慢不行吗？

答案是否定的。

因为 Redis 在不同的软硬件环境下，它的性能是各不相同的。

例如，我的机器配置比较低，当延迟为 2ms 时，我就认为 Redis 变慢了，但是如果你的硬件配置比较高，那么在你的运行环境下，可能延迟是 0.5ms 时就可以认为 Redis 变慢了。

所以，你只有了解了你的 Redis 在生产环境服务器上的基准性能，才能进一步评估，当其延迟达到什么程度时，才认为 Redis 确实变慢了。

具体如何做？

为了避免业务服务器到 Redis 服务器之间的网络延迟，你需要直接在 Redis 服务器上测试实例的响应延迟情况。执行以下命令，就可以测试出这个实例 60 秒内的最大响应延迟：

```shell
$ redis-cli -h 127.0.0.1 -p 6379 --intrinsic-latency 60
Max latency so far: 1 microseconds.
Max latency so far: 15 microseconds.
Max latency so far: 17 microseconds.
Max latency so far: 18 microseconds.
Max latency so far: 31 microseconds.
Max latency so far: 32 microseconds.
Max latency so far: 59 microseconds.
Max latency so far: 72 microseconds.

1428669267 total runs (avg latency: 0.0420 microseconds / 42.00 nanoseconds per run).
Worst run took 1429x longer than the average latency.
```

从输出结果可以看到，这 60 秒内的最大响应延迟为 72 微秒（0.072毫秒）。

你还可以使用以下命令，查看一段时间内 Redis 的最小、最大、平均访问延迟：

```shell
$ redis-cli -h 127.0.0.1 -p 6379 --latency-history -i 1
min: 0, max: 1, avg: 0.13 (100 samples) -- 1.01 seconds range
min: 0, max: 1, avg: 0.12 (99 samples) -- 1.01 seconds range
min: 0, max: 1, avg: 0.13 (99 samples) -- 1.01 seconds range
min: 0, max: 1, avg: 0.10 (99 samples) -- 1.01 seconds range
min: 0, max: 1, avg: 0.13 (98 samples) -- 1.00 seconds range
min: 0, max: 1, avg: 0.08 (99 samples) -- 1.01 seconds range
...
```

以上输出结果是，每间隔 1 秒，采样 Redis 的平均操作耗时，其结果分布在 0.08 ~ 0.13 毫秒之间。

了解了基准性能测试方法，那么你就可以按照以下几步，来判断你的 Redis 是否真的变慢了：

1. 在相同配置的服务器上，测试一个正常 Redis 实例的基准性能
2. 找到你认为可能变慢的 Redis 实例，测试这个实例的基准性能
3. 如果你观察到，这个实例的运行延迟是正常 Redis 基准性能的 2 倍以上，即可认为这个 Redis 实例确实变慢了

确认是 Redis 变慢了，那如何排查是哪里发生了问题呢？

下面跟着我的思路，我们从易到难，一步步来分析可能导致 Redis 变慢的因素。

## 使用复杂度过高的命令

首先，第一步，你需要去查看一下 Redis 的慢日志（slowlog）。

Redis 提供了慢日志命令的统计功能，它记录了有哪些命令在执行时耗时比较久。

查看 Redis 慢日志之前，你需要设置慢日志的阈值。例如，设置慢日志的阈值为 5 毫秒，并且保留最近 500 条慢日志记录：

```shell
# 命令执行耗时超过 5 毫秒，记录慢日志
CONFIG SET slowlog-log-slower-than 5000
# 只保留最近 500 条慢日志
CONFIG SET slowlog-max-len 500
```

设置完成之后，所有执行的命令如果操作耗时超过了 5 毫秒，都会被 Redis 记录下来。

此时，你可以执行以下命令，就可以查询到最近记录的慢日志：

```shell
127.0.0.1:6379> SLOWLOG get 5
1) 1) (integer) 32693       # 慢日志ID
   2) (integer) 1593763337  # 执行时间戳
   3) (integer) 5299        # 执行耗时(微秒)
   4) 1) "LRANGE"           # 具体执行的命令和参数
      2) "user_list:2000"
      3) "0"
      4) "-1"
2) 1) (integer) 32692
   2) (integer) 1593763337
   3) (integer) 5044
   4) 1) "GET"
      2) "user_info:1000"
...
```

通过查看慢日志，我们就可以知道在什么时间点，执行了哪些命令比较耗时。

如果你的应用程序执行的 Redis 命令有以下特点，那么有可能会导致操作延迟变大：

1. 经常使用 O(N) 以上复杂度的命令，例如 SORT、SUNION、ZUNIONSTORE 聚合类命令
2. 使用 O(N) 复杂度的命令，但 N 的值非常大

第一种情况导致变慢的原因在于，Redis 在操作内存数据时，时间复杂度过高，要花费更多的 CPU 资源。

第二种情况导致变慢的原因在于，Redis 一次需要返回给客户端的数据过多，更多时间花费在数据协议的组装和网络传输过程中。

另外，我们还可以从资源使用率层面来分析，如果你的应用程序操作 Redis 的 OPS 不是很大，但 Redis 实例的 **CPU 使用率却很高**，那么很有可能是使用了复杂度过高的命令导致的。

除此之外，我们都知道，Redis 是单线程处理客户端请求的，如果你经常使用以上命令，那么当 Redis 处理客户端请求时，一旦前面某个命令发生耗时，就会导致后面的请求发生排队，对于客户端来说，响应延迟也会变长。

![图片](img\redis001.png)

针对这种情况如何解决呢？

答案很简单，你可以使用以下方法优化你的业务：

1. 尽量不使用 O(N) 以上复杂度过高的命令，对于数据的聚合操作，放在客户端做
2. 执行 O(N) 命令，保证 N 尽量的小（推荐 N <= 300），每次获取尽量少的数据，让 Redis 可以及时处理返回

## 操作bigkey

如果你查询慢日志发现，并不是复杂度过高的命令导致的，而都是 SET / DEL 这种简单命令出现在慢日志中，那么你就要怀疑你的实例否写入了 bigkey。

Redis 在写入数据时，需要为新的数据分配内存，相对应的，当从 Redis 中删除数据时，它会释放对应的内存空间。

如果一个 key 写入的 value 非常大，那么 Redis 在**分配内存时就会比较耗时**。同样的，当删除这个 key 时，**释放内存也会比较耗时**，这种类型的 key 我们一般称之为 bigkey。

此时，你需要检查你的业务代码，是否存在写入 bigkey 的情况。你需要评估写入一个 key 的数据大小，尽量避免一个 key 存入过大的数据。

如果已经写入了 bigkey，那有没有什么办法可以扫描出实例中 bigkey 的分布情况呢？

答案是可以的。

Redis 提供了扫描 bigkey 的命令，执行以下命令就可以扫描出，一个实例中 bigkey 的分布情况，输出结果是以类型维度展示的：

```shell
$ redis-cli -h 127.0.0.1 -p 6379 --bigkeys -i 0.01

...
-------- summary -------

Sampled 829675 keys in the keyspace!
Total key length in bytes is 10059825 (avg len 12.13)

Biggest string found 'key:291880' has 10 bytes
Biggest   list found 'mylist:004' has 40 items
Biggest    set found 'myset:2386' has 38 members
Biggest   hash found 'myhash:3574' has 37 fields
Biggest   zset found 'myzset:2704' has 42 members

36313 strings with 363130 bytes (04.38% of keys, avg size 10.00)
787393 lists with 896540 items (94.90% of keys, avg size 1.14)
1994 sets with 40052 members (00.24% of keys, avg size 20.09)
1990 hashs with 39632 fields (00.24% of keys, avg size 19.92)
1985 zsets with 39750 members (00.24% of keys, avg size 20.03)
```

从输出结果我们可以很清晰地看到，每种数据类型所占用的最大内存 / 拥有最多元素的 key 是哪一个，以及每种数据类型在整个实例中的占比和平均大小 / 元素数量。

其实，使用这个命令的原理，就是 Redis 在内部执行了 SCAN 命令，遍历整个实例中所有的 key，然后针对 key 的类型，分别执行 STRLEN、LLEN、HLEN、SCARD、ZCARD 命令，来获取 String 类型的长度、容器类型（List、Hash、Set、ZSet）的元素个数。

这里我需要提醒你的是，当执行这个命令时，要注意 2 个问题：

1. 对线上实例进行 bigkey 扫描时，Redis 的 OPS 会突增，为了降低扫描过程中对 Redis 的影响，最好控制一下扫描的频率，指定 -i 参数即可，它表示扫描过程中每次扫描后休息的时间间隔，单位是秒
2. 扫描结果中，对于容器类型（List、Hash、Set、ZSet）的 key，只能扫描出元素最多的 key。但一个 key 的元素多，不一定表示占用内存也多，你还需要根据业务情况，进一步评估内存占用情况

那针对 bigkey 导致延迟的问题，有什么好的解决方案呢？

这里有两点可以优化：

- 业务应用尽量避免写入 bigkey

- 如果你使用的 Redis 是 4.0 以上版本，用 UNLINK 命令替代 DEL，此命令可以把释放 key 内存的操作，放到后台线程中去执行，从而降低对 Redis 的影响

- 如果你使用的 Redis 是 6.0 以上版本，可以开启 lazy-free 机制（lazyfree-lazy-user-del = yes），在执行 DEL 命令时，释放内存也会放到后台线程中执行

但即便可以使用方案 2，我也不建议你在实例中存入 bigkey。

这是因为 bigkey 在很多场景下，依旧会产生性能问题。例如，bigkey 在分片集群模式下，对于数据的迁移也会有性能影响，以及我后面即将讲到的数据过期、数据淘汰、透明大页，都会受到 bigkey 的影响。

## 集中过期

