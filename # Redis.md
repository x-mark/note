# Redis
## 功能
提供key-value内存数据库功能：
1. 支持持久化
2. 数据类型：k-v,list,set,zset(sort set),hash
3. master-slave模式数据备份

## 优点
1. 读写性能极高。

2. 数据类型丰富。

3. 原子操作，单个操作原子特性，多操作支持事务（MULTI开始，EXEC结束）。

4. BSD协议。  [开源协议介绍](http://www.runoob.com/w3cnote/open-source-license.html)


## redis和memcached区别

|区别|Redis|Memcached|
|---|---|---|
|网络IO模型|单线程IO复用,线程池|非阻塞IO复用，Listen+workers|
|数据支持类型|k-v，list,set,zset,hash|k-v|
|内存管理机制|现场申请|Slab Allocation,不同大小chunk，无碎片，有浪费|
|数据存储及持久化|可持久化|in-memory|
|数据一致性|事务机制|cas命令保证并发访问|
|集群管理|服务端构建，线性可伸缩|客户端实现|