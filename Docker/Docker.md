# Docker相比于虚拟机

Linux发展出了一种新的虚拟化技术：Linux Containers （LXC）。

LXC不需要模拟一个完整的操作系统，而是对进程进行隔离。

Docker优势体现为启动速度快、占用内存小。

Docker可实现更快速的应用交付与部署，更便捷的升级和扩缩容，更简单的系统运维，更高效的计算资源利用。

|            | Docker容器              | 虚拟机                      |
| ---------- | ----------------------- | --------------------------- |
| 操作系统   | 与宿主机共享OS          | 宿主机OS上运行虚拟机OS      |
| 存储大小   | 镜像小                  | 镜像庞大（vmdk、vdi等）     |
| 运行性能   | 几乎无额外性能损失      | 操作系统额外的CPU、内存消耗 |
| 移植性     | 轻便、灵活，适应与Linux | 笨重                        |
| 硬件亲和性 | 面向软件开发者          | 面向硬件运维者              |
| 部署速度   | 快速，秒级              | 较慢，10s以上               |



## docker有着比虚拟机更少的抽象层

docker不需要Hypervisor（虚拟机）实现硬件资源虚拟化，运行在docker容器上的程序直接使用的都是实际物理机的硬件资源。

因此，Docker在CPU、内存利用率上有着明显的优势。

## docker利用的是宿主机的内存，而不需要加载操作系统OS内核

避免引导、加载操作系统内核返回等比较费时费资源的过程。

# Docker的卸载

[Docker官网Unstall教程](https://docs.docker.com/engine/install/)

```bash
systemctl stop docker
yum remove docker-ce docker-ce-cli containerd.io docker-compose-plugin
rm -rf /var/lib/docker
rm -rf /var/lib/containerd
```

# Docker的安装

[Docker官网Install教程](https://docs.docker.com/engine/install/)

## 注意：安装Docker之前需要搭建Linux gcc 环境。

```bash
yum -y install gcc
yum -y install gcc-c++
```

## 安装yum工具

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2 --skip-broken
```

## 设置docker镜像工具

```shell
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

执行如下命令将docker-ce.repo镜像仓库配置文件中的镜像源修改成阿里镜像源：

```shell
sed -i 's/download.docker.com/mirrors.aliyun.com\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo
```

## 注意：建议更新一下yum软件包索引

```bash
yum makecache fast
```

## 安装docker

```shell
yum -y install docker-ce docker-ce-cli containerd.io
```

## 验证docker安装成功，查看Docker版本

```bash
docker version
```

## 启动docker

```bash
systemctl start docker
```

## 设置docker开机自动启动

```shell
systemctl enable docker
```

## docker run hello-world

```bash
docker run hello-world
```

## 阿里云-容器镜像服务

[登录阿里云搜索：容器镜像服务](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors?accounttraceid=e5e6948c0f394fd3941307510891df6coeqo)

注意下方，registry-mirrors 参数需要修改，参考阿里云镜像连接

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxxxx.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

# 安装docker-compose环境

```shell
curl -SL https://github.com/docker/compose/releases/download/v2.17.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
```

当然可以先下载下来，在上传到服务器，毕竟服务器下载很慢（github）。

给docker-compose添加可执行权限：

```shell
sudo chmod +x /usr/local/bin/docker-compose
```

下载并安装成功后，使用如下命令创建docker-compose软链接

```shell
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

随后查看docker-compose版本：

```shell
docker-compose version
```

成功安装：

```shell
Docker Compose version v2.17.3
```

使用（注意用自己的yml文件）：

```shell
docker-compose -f <docker-compose-xxxxx.yml> up -d --force-recreate
```

关闭启动的容器

```shell
docker-compose -f <docker-compose-xxxxx.yml> down
```

# Docker 常用命令

## 启动Docker

```bash
systemctl start docker
```

## 停止Docker

```bash
systemctl stop docker
```

## 重启Docker

```bash
systemctl restart docker
```

## 查看Docker状态

```bash
systemctl status docker
```

## 开机自动启动

```bash
systemctl enable docker
```

## 查看Docker概要信息

```bash
docker info
```

## Docker帮助文档

```bash
docker --help
```

## 查看所有local镜像

```bash
docker images
```

## 搜索镜像

```bash
docker search <images_name>
docker search --limit 5 redis
```

## 下载镜像

```bash
docker pull <images_name> [:tag]
```

## 查看镜像/容器/数据卷所占的空间

```bash
docker system df
```

## 删除某个镜像

```bash
docker rmi <imgae_name/image_id>
## 强制删除
docker rmi <imgae_name/image_id>
## 查询删除，删除 $ 里面的查询出来的镜像
docker rmi $(docker images -qa)
## 举例
docker rmi hello-world
```

## 启动镜像

```bash
docker run <image_name>
## 参数i代表interactive  t代表terminal(tty)  --name 指定容器名
docker run -it --name=<container_name> <image_name>
```

## 查看所有容器

```shell
docker ps -a
```

## 列出当前正在运行的容器

```bash
docker ps
```

## 退出容器-容器停止

```bash
exit
```

## 退出容器-容器不停止

```bash
ctrl+p+q
```

## 容器的启动与停止

```bash
## 启动已停止的容器
docker start <container_id>
## 重启容器
docker restart <container_id>
## 停止容器
docker stop <container_id>
## 强制停止容器
docker kill <container_id>
## 删除容器
docker rm <container_id>
## 强制删除容器
docker rm -f <container_id>
## 一次性删除多个docker容器
docker rm -f $(docker ps -a -q)
docker ps -a -q | xargs docker rm
```

