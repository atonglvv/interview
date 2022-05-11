# SqlSession默认自动提交事务么？

SqlSession默认不自动提交事务，可以使用 `SqlSessionFactory.openSession(true)` 开启自动提交。

# Mybatis核心配置文件xml，标签的顺序？

# Mybatis获取参数值的两种方式

${} : 字符串拼接，会造成sql注入，并且需要注意单引号问题。

#{}：占位符赋值。

# 用过@MapKey注解么？



# 模糊查询怎么做？用 ${} 还是 #{} ？

**方法一：**

```sql
like concat('%', #{}, '%')
```

**方法二（建议）：**

```sql
like "%" #{} "%"
```



# 动态设置表名，将表名作为入参

需要使用 ${}

```sql
select * from ${tableName}
```



# 多对一查询resultMap怎么写？

association：处理多对一映射关系

# 一对多查询怎么处理？

collection：处理一对多映射关系

# mybatis的延迟加载

延迟加载适用于多表关联查询的业务场景。

使用延迟加载可以避免表连接查询，表连接查询比单表查询的效率低，但是它需要多次与数据库进行交互，所以延迟加载并不是银弹，使用需谨慎。

# 动态标签

## trim

prefix：表示在trim标签包裹的部分的前面添加内容（拼接前缀）。

suffix：表示在trim标签包裹的部分的后面添加内容（拼接后缀）。

prefixOverrides：去除sql语句前面的关键字或者字符，该关键字或者字符由prefixOverrides属性指定，假设该属性指定为"and"，当sql语句的开头为"and"，trim标签将会去除该"and"。

suffixOverrides：去除sql语句后面的关键字或者字符，该关键字或者字符由suffixOverrides属性指定。



# Mybatis的缓存

## Mybatis的一级缓存

默认开启。

SqlSession级别。

### 如何清空一级缓存？

```java
sqlSession.clearCache();
```

## Mybatis的二级缓存

SqlSessionFactory级别。







































