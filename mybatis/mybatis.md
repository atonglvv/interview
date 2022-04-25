# SqlSession默认自动提交事务么？

SqlSession默认不自动提交事务，可以使用 `SqlSessionFactory.openSession(true)` 开启自动提交。

# Mybatis核心配置文件xml，标签的顺序？

# Mybatis获取参数值的两种方式

${} : 字符串拼接，会造成sql注入，并且需要注意单引号问题。

#{}：占位符赋值。