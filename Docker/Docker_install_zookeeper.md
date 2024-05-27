# 下载最新版镜像

```shell
 docker search zookeeper    
 docker pull zookeeper 
 docker images              //查看下载的本地镜像
 docker inspect zookeeper   //查看zookeeper详细信息
```

# 新建文件夹用于挂载

在 `/usr/local` 下新建`zookeeper`文件夹。

# 启动命令

```shell
docker run -d -e TZ="Asia/Shanghai" -p 2181:2181 -v /usr/local/zookeeper:/data --name zookeeper --restart always zookeeper

```

# 参数解释

```shell
 -e TZ="Asia/Shanghai" # 指定上海时区 
 -d # 表示在一直在后台运行容器
 -p 2181:2181 # 对端口进行映射，将本地2181端口映射到容器内部的2181端口
 --name # 设置创建的容器名称
 -v # 将本地目录(文件)挂载到容器指定目录；
 --restart always #始终重新启动zookeeper
```

# 查看容器

```shell
docker ps
```

# 进入zookeeper客户端

```shell
 docker exec -it zookeeper bash  ./bin/zkCli.sh
```

