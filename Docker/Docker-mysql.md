# 拉取Docker最近版本镜像

```dockerfile
docker pull mysql:8.4
```



# 运行容器

## 8.0

```shell
docker run \
--name mysql \
--privileged=true \
-d \
-p 3306:3306 \
-v ~/soft/mysql/log:/var/log/mysql \
-v ~/soft/mysql/data:/var/lib/mysql \
-v ~/soft/mysql/conf:/etc/mysql/conf.d \
-v ~/soft/mysql/mysql-files:/var/lib/mysql-files \
-e MYSQL_ROOT_PASSWORD=1230 \
mysql:8.0
```





## 8.4

```dockerfile
docker run \
--name mysql \
--privileged=true \
-d \
-p 3306:3306 \
-v ~/soft/mysql/log:/var/log/mysql \
-v ~/soft/mysql/data:/var/lib/mysql \
-v ~/soft/mysql/conf:/etc/mysql/conf.d \
-v ~/soft/mysql/mysql-files:/var/lib/mysql-files \
-e MYSQL_ROOT_PASSWORD=1230 \
mysql:8.4
```

## 遇到的错误

```shell
2025-05-02 08:12:05+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.3.0-1.el9 started.


2025-05-02 08:12:05+00:00 [ERROR] [Entrypoint]: mysqld failed while attempting to check config


command was: mysqld --verbose --help --log-bin-index=/tmp/tmp.hUQA9d0y3I


mysqld: Can't read dir of '/etc/mysql/conf.d/' (OS errno 2 - No such file or directory)


mysqld: [ERROR] Stopped processing the 'includedir' directive in file /etc/my.cnf at line 32.


mysqld: [ERROR] Fatal error in defaults handling. Program aborted!
```

### 原因分析

**原因**：这意味着 MySQL 在启动时无法找到 /etc/mysql/conf.d/ 目录，mysql容器的/etc/mysql目录挂载到宿主机的 /Users/atonglv/soft/mysql/conf目录，这通常是因为这个目录在你挂载的配置卷 /Users/atonglv/soft/mysql/conf 中不存在或没有正确创建，导致容器创建失败。

### 解决

检查本地配置目录：检查宿主机的 /Users/atonglv/soft/mysql/conf 目录中是否存在 `conf.d` 和 `mysql.conf.d` 子目录，如果不存在，创建这个子目录。

```shell
# 创建子目录
mkdir conf.d
mkdir mysql.conf.d
```



```sql
CREATE USER 'atong'@'%' IDENTIFIED WITH 'mysql_native_password' BY '1230';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'atong'@'%';
FLUSH PRIVILEGES;
```





MySQL 8.4默认使用`caching_sha2_password` 插件进行身份验证。需要对验证方式进行修改。修改为`mysql_native_password`。

```sql
ALTER USER 'root'@'%' IDENTIFIED WITH 'mysql_native_password' BY '123!@#qwe';
ALTER USER 'atong'@'%' IDENTIFIED WITH 'mysql_native_password' BY '123!@#qwe';
ALTER USER 'root'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY '123!@#qwe';
FLUSH PRIVILEGES;
```

