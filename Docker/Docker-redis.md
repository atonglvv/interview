



# 运行脚本

```shell
docker run --name redis -d \
-p 6379:6379 \
-v ~/soft/redis/conf:/etc/redis \
-v ~/soft/redis/data:/data \
redis \
redis-server /etc/redis/redis.conf
```

