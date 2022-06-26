# Docker相比于虚拟机

Linux发展出了一种新的虚拟化技术：Linux Containers （LXC）。

LXC不需要模拟一个完整的操作系统，而是对进程进行隔离。

Docker优势体现为启动速度快、占用内存小。

Docker可实现更快速的应用交付与部署，更便捷的升级和扩缩容，更简单的系统运维，更高效的计算资源利用。



# Docker的安装

[Docker官网Install教程](https://docs.docker.com/engine/install/)

## 注意：安装Docker之前需要搭建Linux gcc 环境。

```bash
yum -y install gcc
yum -y install gcc-c++
```

## 注意：Set up the repository

不要安装官网推荐的 repository。容易Timeout。

推荐阿里云repository。

## 注意：建议更新一下yum软件包索引

```bash
yum makecache fast
```

## 验证docker安装成功

```bash
docker version
```

## docker run hello-world

```bash
docker run hello-world
```

# Docker的卸载

[Docker官网Unstall教程](https://docs.docker.com/engine/install/)

```bash
systemctl stop docker
yum remove docker-ce docker-ce-cli containerd.io docker-compose-plugin
rm -rf /var/lib/docker
rm -rf /var/lib/containerd
```





