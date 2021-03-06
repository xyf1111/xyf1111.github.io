# Redis Study 01

<!--more-->
## NoSQL的四大分类
### KV键值对
- 新浪：Redis
- 美团：Redis+Tair
- 阿里、百度：Redis+Memecache
### 文档型数据库
- MongoDB（一般必须掌握）
    - MongoDB是一个基于分布式文件存储的数据库，C++编写，用来处理大量文档
    - MongoDB是一个介于关系型数据库和非关系型数据库之间的产品，MongoDB是非关系型数据库中功能最丰富的，最像关系型数据库的
- ConthDB（国外，不需了解）
### 列存储数据库
- HBase
- 分布式文件系统
### 图关系数据库
- 不是存放图形，是存放关系的
- Neo4j、infoGrid
## Redis概述
### Redis是什么
Redis（Remote Dictionary Server）远程字典服务
是一个开源的使用C语言编写、支持网络，可基于内存亦可持久化的日志型，Key-Value的数据库，提供多种语言的API
redis会周期性的把更新的数据写进磁盘或者把修改操作写入追加的记录文件，并在此基础上实现master-slave（主从）同步
### Redis能干嘛
1. 内存存储，持久化，内存中是断点即失的，所以说持久化很重要（rdb，aof）
2. 效率高，可以用于高速缓存
3. 发布订阅系统
4. 地图信息分析
5. 计时器，计数器
。。。
### Redis特性
1. 多样的数据类型
2. 持久化
3. 集群
4. 事务
。。。
## 测试Redis性能
### 使用benchmark性能测试
使用Redis-benchmark性能测试
## Redis基础知识
### Redis基本操作命令
- Redis默认有16个数据库
- 默认使用第0个数据库
- select index，切换数据库
- DBSIZE，查看当前数据库大小
- keys *，查看当前数据库所有的key
- flushdb，清除当前数据库
- flushall，清除所有数据库
- set name cc，set key
- exists name，判断当前的key是否存在
- move name，移除当前的key
- expire name 10，设置key的过期时间单位秒
- ttl name，查看key的剩余时间
- get name，获得key
- type name，查看key的类型
### Redis是单线程的
Redis是很快的。官方表示Redis是基于内存操作的，CPU并不是Redis的瓶颈Redis是根据机器的内存和网络带宽，可以使用单线程就使用单线程。
### Redis为什么单线程还这么快
1. 误区1：高性能的服务器一定是多线程的
2. 误区2：多线程（CPU上下文切换）一定比单线程效率高
核心，Redis是将所有的数据全部存放在内存中，所以使用单线程去操作效率就是最高的。多线程（CPU上下文切换：耗时操作），对于内存系统来说，没有上下文切换效率就是最高的。多次读写都是在一个CPU上，在内存情况下，这个就是最优的方案。
