# EasyExcel

## 为什么要使用EasyExcel

Java解析Excel的工具有很多，poi、jxl，但存在如下问题：

- 非常消耗内存

easyExcel优点：

- 遇到再大的excel都不会出现内存溢出的问题



## EasyExcel拟定解决的问题

excel读写时内存溢出的问题

使用简单

excel格式解析



## 工作原理



Excel解析流程：



xls文件解压缩  -->  SAX模式读取xml文件  -->  模型转换  -->  返回给调用者

