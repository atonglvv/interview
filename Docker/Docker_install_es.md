# Elastic官网文档

https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html

# pull image

```shell
sudo docker pull elasticsearch:7.12.0
```

# 创建es挂在目录

```shell
sudo mkdir -p /opt/elasticsearch/config
sudo mkdir -p /opt/elasticsearch/data
sudo mkdir -p /opt/elasticsearch/plugins
```

# 配置文件

```shell
echo "http.host: 0.0.0.0" >> /opt/elasticsearch/config/elasticsearch.yml
```

# docker run

```shell
  docker run -d \
  --name=elasticsearch \
  --restart=always \
  -p 9200:9200 \
  -p 9300:9300 \
  -v /opt/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
  -v /opt/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
  -v /opt/elasticsearch/data:/usr/share/elasticsearch/data \
  -e "discovery.type=single-node" \
  -e "ES_JAVA_OPTS=-Xms128m -Xmx256m" \
  elasticsearch:7.12.0
```

**说明:**

- -d 后台运行
- --name 容器名
- -p 端口映射，9200是http请求，restfulAPI用，9300是es在集群环境下节点通讯端口
- -e discovery.type=single-node 单点模式启动
- -e "ES_JAVA_OPTS=-Xms128m -Xmx256m"：设置启动占用的内存范围
- -v 目录挂载

# 测试安装成功

浏览器访问：ip:9200

发现安装失败。。

# 查看日志

```shell
docker logs elasticsearch
```

```bash
ElasticsearchException[failed to bind service]; nested: AccessDeniedException[/usr/share/elasticsearch/data/nodes];
Likely root cause: java.nio.file.AccessDeniedException: /usr/share/elasticsearch/data/nodes
	at java.base/sun.nio.fs.UnixException.translateToIOException(UnixException.java:90)
	at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:106)
	at java.base/sun.nio.fs.UnixException.rethrowAsIOException(UnixException.java:111)
	at java.base/sun.nio.fs.UnixFileSystemProvider.createDirectory(UnixFileSystemProvider.java:388)
	at java.base/java.nio.file.Files.createDirectory(Files.java:694)
	at java.base/java.nio.file.Files.createAndCheckIsDirectory(Files.java:801)
	at java.base/java.nio.file.Files.createDirectories(Files.java:787)
	at org.elasticsearch.env.NodeEnvironment.lambda$new$0(NodeEnvironment.java:265)
	at org.elasticsearch.env.NodeEnvironment$NodeLock.<init>(NodeEnvironment.java:202)
	at org.elasticsearch.env.NodeEnvironment.<init>(NodeEnvironment.java:262)
	at org.elasticsearch.node.Node.<init>(Node.java:352)
	at org.elasticsearch.node.Node.<init>(Node.java:278)
	at org.elasticsearch.bootstrap.Bootstrap$5.<init>(Bootstrap.java:217)
	at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:217)
	at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:397)
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:159)
	at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:150)
	at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:75)
	at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:116)
	at org.elasticsearch.cli.Command.main(Command.java:79)
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:115)
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:81)
For complete error details, refer to the log at /usr/share/elasticsearch/logs/elasticsearch.log
```

发现是没有权限。

# 授权

```bash
chmod -R 777 /opt/elasticsearch/
```

# 重启

```shell
docker restart <container_id>
```

<container_id> 为 es 的 容器Id。

# 测试安装成功

ip:port

```json
{
  "name" : "f22060c39996",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "RINHqU4QSPa8R054wtFl0g",
  "version" : {
    "number" : "7.12.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "78722783c38caa25a70982b5b042074cde5d3b3a",
    "build_date" : "2021-03-18T06:17:15.410153305Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

