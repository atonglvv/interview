# 慢sql

[`https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html`](https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html)

## 打开慢日志开关

因为开启慢查询日志是有代价的（跟 binlog、optimizer-trace 一样），所以它默认是关闭的：

```sql
show variables like 'slow_query%';
```

![](img\慢sql01.png)	

除了这个开关，还有一个参数，控制执行超过多长时间的 SQL 才记录到慢日志，默认是 10 秒。

```sql
show variables like '%long_query%';
可以直接动态修改参数（重启后失效）。
set @@global.slow_query_log=1; -- 1 开启，0 关闭，重启后失效 
set @@global.long_query_time=3; -- mysql 默认的慢查询时间是 10 秒，另开一个窗口后才会查到最新值 

show variables like '%long_query%'; 
show variables like '%slow_query%';
或者修改配置文件 my.cnf。
以下配置定义了慢查询日志的开关、慢查询的时间、日志文件的存放路径。
slow_query_log = ON 
long_query_time=2 
slow_query_log_file =/var/lib/mysql/localhost-slow.log
模拟慢查询：
select sleep(10);
查询 user_innodb 表的 500 万数据（检查是不是没有索引）。
SELECT * FROM `user_innodb` where phone = '136';
```

## 慢日志分析

**1、日志内容**

```sql
show global status like 'slow_queries'; -- 查看有多少慢查询 
show variables like '%slow_query%'; -- 获取慢日志目录
cat /var/lib/mysql/ localhost-slow.log
```

![](img\慢sql02.png)	

```
有了慢查询日志，怎么去分析统计呢？比如 SQL 语句的出现的慢查询次数最多，平均每次执行了多久？人工肉眼分析显然不可能。
```

**2、mysqldumpslow**

```
https://dev.mysql.com/doc/refman/5.7/en/mysqldumpslow.html
```

MySQL 提供了 mysqldumpslow 的工具，在 MySQL 的 bin 目录下。

```sql
mysqldumpslow --help
```

例如：查询用时最多的 10 条慢 SQL：

```sql
mysqldumpslow -s t -t 10 -g 'select' /var/lib/mysql/localhost-slow.log
```

![](img\慢sql03.png)	

Count 代表这个 SQL 执行了多少次；

Time 代表执行的时间，括号里面是累计时间；

Lock 表示锁定的时间，括号是累计；

Rows 表示返回的记录数，括号是累计。

除了慢查询日志之外，还有一个 SHOW PROFILE 工具可以使用。