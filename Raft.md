简易版Raft分布式共识算法
最近学习raft算法，写了一个简易版Raft分布式共识算法的kv存储，实现了选举、心跳、日志同步、日志校验、间接性的实现了一致性读，没有实现快照和压缩:

项目地址:[raft-java-demo: 用iava实现的一个简易版Raft共识算法，实现了一个简易版KV存储...](https://gitee.com/colins0902/raft-java-demo)(不完全参照Raft论文实现，做出了自己的一些改动，核心思想不变，测试类、注释完善，当然可能也存在问题，欢迎提出)
文档地址: [raft-java-demo ReturnAC](https://returnac.cn/pages/wheel/Raft-java-demo.html)
参考资料:Raft一致性算法论文-译文:  [raft-zh cn/raft-zh cn.md at master' maemual/raft-...](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)

Raft动画演示地址: [Raft](http://thesecretlivesofdata.com/raft/)
