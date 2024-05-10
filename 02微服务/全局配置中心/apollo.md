# 官方文档

https://www.apolloconfig.com/#/zh/design/apollo-introduction

# 架构剖析

https://mp.weixin.qq.com/s/-hUaQPzfsl9Lm3IqQW3VDQ

## 核心模块

### ConfigService

提供配置获取接口

提供配置推送接口

服务于Client

### AdminService

提供配置管理接口

提供配置修改发布接口

服务于管理界面Portal

### Client

为应用获取配置，支持实时更新

通过MetaServer获取ConfigService的服务列表

使用客户端软负载SLB方式调用ConfigService

### Portal

配置管理界面

存放用户权限、项目和配置的元数据信息

通过MetaServer获取AdminService的服务列表

使用客户端软负载SLB方式调用AdminService

## 辅助模块

### Eureka

用于服务发现和注册

Config/AdminService注册实例并定期报心跳

和ConfigService在一起部署

### MetaServer



Portal通过域名访问MetaServer获取AdminService的地址列表

Client通过域名访问MetaServer获取ConfigService的地址列表

相当于一个Eureka Proxy，将Eureka的服务发现接口以更简单明确的HTTP接口的形式暴露出来，方便Client/Protal通过简单的HTTPClient就可以查询到Config/AdminService的地址列表。

逻辑角色，和ConfigService住在一起部署

### NginxLB

和域名系统配合，协助Portal访问MetaServer获取AdminService地址列表

和域名系统配合，协助Client访问MetaServer获取ConfigService地址列表

和域名系统配合，协助用户访问Portal进行配置管理



# 问题

## 为什么需要MetaServer？

在携程，应用场景不仅有Java，还有很多遗留的.Net应用。Apollo的作者也考虑到开源到社区以后，很多客户应用是非Java的。但是Eureka(包括Ribbon软负载)原生仅支持Java客户端，如果要为多语言开发Eureka/Ribbon客户端，这个工作量很大也不可控。为此，Apollo的作者引入了MetaServer这个角色，它其实是一个Eureka的Proxy，将Eureka的服务发现接口以更简单明确的HTTP接口的形式暴露出来，方便Client/Protal通过简单的HTTPClient就可以查询到Config/AdminService的地址列表。获取到服务实例地址列表之后，再以简单的客户端软负载(Client SLB)策略路由定位到目标实例，并发起调用。

## MetaServer本身也是无状态以集群方式部署的，那么Client/Protal该如何发现MetaServer呢？

一种传统的做法是借助硬件或者软件负载均衡器，例如在携程采用的是扩展后的NginxLB（也称Software Load Balancer），由运维为MetaServer集群配置一个域名，指向NginxLB集群，NginxLB再对MetaServer进行负载均衡和流量转发。Client/Portal通过域名+NginxLB间接访问MetaServer集群。

## Portal也是无状态以集群方式部署的，用户如何发现和访问Portal？

用户通过域名+NginxLB间接访问Portal集群。







1. 