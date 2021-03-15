# Redis Study 02

<!--more-->
## 五大数据类型
### Stirng（字符串）
```bash
#######################################################
#   set
#   get
#   exists
#   append
127.0.0.1:6379> set key1 v1 # 设置key1
OK
127.0.0.1:6379> get key1 # 获得值
"v1"
127.0.0.1:6379> exists key1 # 判断key是否存在
(integer) 1
127.0.0.1:6379> append key1 "hello" # 追加字符串，如果当前key不存在相当于set key
(integer) 7
127.0.0.1:6379> get key1
"v1hello"
#######################################################
#   strlen
127.0.0.1:6379> strlen key1 # 获取字符串长度
(integer) 7
127.0.0.1:6379> append key1 ",cc"
(integer) 10
127.0.0.1:6379> strlen key1
(integer) 10
127.0.0.1:6379> get key1
"v1hello,cc"
#######################################################
#   步长 i++
127.0.0.1:6379> incr views # 自增1
(integer) 1
127.0.0.1:6379> get views
"1"
127.0.0.1:6379> decr views # 自减1
(integer) 0
127.0.0.1:6379> incrby views 10 # 设置步长，指定增量
(integer) 10
127.0.0.1:6379> get views
"10"
127.0.0.1:6379> DECRBY views 5
(integer) 5
127.0.0.1:6379> GET views
"5"
#######################################################
# 字符串范围
127.0.0.1:6379> set key1 "hello,cc"
OK
127.0.0.1:6379> GETRANGE key1 0 3 # 截取字符串， [0,3]
"hell"
127.0.0.1:6379> GETRANGE key1 0 -1 # 获取全部字符串，相当于get key
"hello,cc"
# 替换
127.0.0.1:6379> set key2 abcdefg
OK
127.0.0.1:6379> GET key2
"abcdefg"
127.0.0.1:6379> SETRANGE key2 1 xx # 替换指定位置开始的字符串
(integer) 7
127.0.0.1:6379> GET key2
"axxdefg"
#######################################################
# setex (set with expire) 设置过期时间
# setnx (set if not exist) 不存在才设置（在分布式锁中常使用）
127.0.0.1:6379> SETEX key3 30 "hello" # 设置key3为hello，30秒后过期
OK
127.0.0.1:6379> ttl key3 # 查看剩余时间
(integer) 22
127.0.0.1:6379> get key3
"hello"
127.0.0.1:6379> SETNX mykey redis # 如果mykey不存在，创建mykey
(integer) 1
127.0.0.1:6379> get mykey
"redis"
127.0.0.1:6379> SETNX mykey mongodb # 如果mykey存在，创建失败
(integer) 0
127.0.0.1:6379> get mykey
"redis"
127.0.0.1:6379> ttl key3
(integer) -2
127.0.0.1:6379> get key3
(nil)
#######################################################
# mset  （同时设置多个值）
# mget  （同时获取多个值）
127.0.0.1:6379[1]> mset k1 v1 k2 v2 k3 v3 # 同时设置多个值
OK
127.0.0.1:6379[1]> keys *
1) "k1"
2) "k3"
3) "k2"
127.0.0.1:6379[1]> mget k1 k2 k3 # 同时获取多个值
1) "v1"
2) "v2"
3) "v3"
127.0.0.1:6379[1]> mset k1 v1 k4 v4
OK
127.0.0.1:6379[1]> keys *
1) "k1"
2) "k4"
3) "k3"
4) "k2"
127.0.0.1:6379[1]> mget k1 k2 k3 k4
1) "v1"
2) "v2"
3) "v3"
4) "v4"
127.0.0.1:6379[1]> msetnx k1 v1 k5 v5 # msetnx是一个原子性操作，要么都成功，要么都失败
(integer) 0
127.0.0.1:6379[1]> keys *
1) "k1"
2) "k4"
3) "k3"
4) "k2"
#######################################################
# 对象
127.0.0.1:6379[1]> set user:1 {name:zhangsan,age:3} # 设置一个user:1对象，值为json字符来保存一个对象
OK
127.0.0.1:6379[1]> mset user:1:name zhangsan user:1:age 3 # 这里的key是一个巧妙的设计：user:{id}:{filed}，如此设计在redis中是完全ok的
OK
127.0.0.1:6379[1]> mget user:1:name user:1:age
1) "zhangsan"
2) "3"
#######################################################
# getset 先get后set
127.0.0.1:6379[1]> GETSET db redis # 如果不存在值，返回nil，并设置值
(nil)
127.0.0.1:6379[1]> get db
"redis"
127.0.0.1:6379[1]> GETSET db mongodb # 如果存在值，返回原来的值，并做更新操作
"redis"
127.0.0.1:6379[1]> get db
"mongodb"
```
### List
列表是一种基本的数据类型

在Redis里面，我们可以把list完成，栈、队列、堵塞队列。
```bash
#######################################################
#   lpush
#   lpop
#   lrange
127.0.0.1:6379[1]> LPUSH list one # 将一个值或多个值，插入到列表头部
(integer) 1
127.0.0.1:6379[1]> LPUSH list two
(integer) 2
127.0.0.1:6379[1]> LPUSH list three
(integer) 3
127.0.0.1:6379[1]> LRANGE list 0 -1 # 获取list中的值
1) "three"
2) "two"
3) "one"
127.0.0.1:6379[1]> LRANGE list 0 1 # 通过区间获取具体的值
1) "three"
2) "two"
127.0.0.1:6379[1]> RPUSH list right # 将一个或多个值放到列表的尾部
(integer) 4
127.0.0.1:6379[1]> LRANGE list 0 -1
1) "three"
2) "two"
3) "one"
4) "right"
127.0.0.1:6379[1]> LPOP list # 移除列表的第一个元素
"three"
127.0.0.1:6379[1]> RPOP list # 移除列表的最后一个元素
"right"
127.0.0.1:6379[1]> LRANGE list 0 -1
1) "two"
2) "one"
#######################################################
#   lindex
#   llen
127.0.0.1:6379[1]> LINDEX list 0 # 通过下标获取list中的某一个值
"two"
127.0.0.1:6379[1]> LINDEX list 1
"one"
127.0.0.1:6379> LPUSH list one
(integer) 1
127.0.0.1:6379> LPUSH list two
(integer) 2
127.0.0.1:6379> LPUSH list three
(integer) 3
127.0.0.1:6379> LLEN list # 返回列表的长度
(integer) 3
#######################################################
#   lrem
127.0.0.1:6379> LRANGE list 0 -1
1) "three"
2) "three"
3) "two"
4) "one"
127.0.0.1:6379> LREM list 1 one # 移除list集合中指定个数的value
(integer) 1
127.0.0.1:6379> LRANGE list 0 -1
1) "three"
2) "three"
3) "two"
127.0.0.1:6379> LREM list 2 three
(integer) 2
127.0.0.1:6379> LRANGE list 0 -1
1) "two"
#######################################################
#   trim修剪
127.0.0.1:6379> RPUSH mylist hello
(integer) 1
127.0.0.1:6379> RPUSH mylist hello1
(integer) 2
127.0.0.1:6379> RPUSH mylist hello2
(integer) 3
127.0.0.1:6379> RPUSH mylist hello3
(integer) 4
127.0.0.1:6379> LRANGE mylist 0 -1
1) "hello"
2) "hello1"
3) "hello2"
4) "hello3"
127.0.0.1:6379> LTRIM mylist 1 2    # 通过下标截取指定长度，这个list已经改变，只剩下截取的元素
OK
127.0.0.1:6379> LRANGE mylist 0 -1
1) "hello1"
2) "hello2"
#######################################################
#   rpoplpush
127.0.0.1:6379> LRANGE mylist 0 -1
1) "hello1"
2) "hello2"
127.0.0.1:6379> RPOPLPUSH mylist myotherlist # 移动列表的最后一个元素到其他表中
"hello2"
127.0.0.1:6379> LRANGE mylist 0 -1 # 查看原来的表
1) "hello1"
127.0.0.1:6379> LRANGE myotherlist 0 -1 # 查看新的表
1) "hello2"
#######################################################
#   exists
#   lset    将列表指定下标的值，替换为另外一个值，更新操作
127.0.0.1:6379> exists list # 判断列表是否存在
(integer) 0
127.0.0.1:6379> lset list 0 item # 不存在，更新，报错
(error) ERR no such key
127.0.0.1:6379> LPUSH list value
(integer) 1
127.0.0.1:6379> lset list 0 item # 存在，更新当前下标的值
OK
127.0.0.1:6379> LRANGE list 0 -1
1) "item"
127.0.0.1:6379> lset list 1 other # 列表不存在，也报错
(error) ERR index out of range
#######################################################
#   linsert 将某个具体的value插入列表中某个元素的前面或者后面
127.0.0.1:6379> LPUSH mylist hello
(integer) 1
127.0.0.1:6379> LPUSH mylist world
(integer) 2
127.0.0.1:6379> LINSERT mylist before world other
(integer) 3
127.0.0.1:6379> LRANGE mylist 0 -1
1) "other"
2) "world"
3) "hello"
127.0.0.1:6379> LINSERT mylist after world other1
(integer) 4
127.0.0.1:6379> LRANGE mylist 0 -1
1) "other"
2) "world"
3) "other1"
4) "hello"
```
小结：
- list实际上是一个链表，before Node after，left，right都可以插入
- 如果key不存在，创建新的链表
- 如果key存在，新增内容
- 如果移除了key，空链表，也代表不存在
- 在两边插入或改动值，效率最高。中间元素，相对来说效率会低一点
使用场景：
- 消息排队
- 消息队列
### Set（集合）
Set中的值是不能重复的
```bash
#######################################################
#   sadd
#   smembers
#   sismember
127.0.0.1:6379> sadd myset "hello"  # set集合中添加元素
(integer) 1
127.0.0.1:6379> sadd myset cc
(integer) 1
127.0.0.1:6379> SMEMBERS myset  # 查看指定set所有值
1) "cc"
2) "hello"
127.0.0.1:6379> SISMEMBER myset cc  # 判断某一个值在不在set中
(integer) 1
127.0.0.1:6379> SISMEMBER myset a
(integer) 0
#######################################################
#   scard   # 获取set集合中的内容元素个数
127.0.0.1:6379> scard myset # 获取set集合元素个数
(integer) 2
127.0.0.1:6379> SMEMBERS myset
1) "cc"
2) "hello"
127.0.0.1:6379> SADD myset cc   # set 已存在元素
(integer) 0
127.0.0.1:6379> SADD myset cc1
(integer) 1
127.0.0.1:6379> SCARD myset
(integer) 3
127.0.0.1:6379> SMEMBERS myset
1) "cc"
2) "hello"
3) "cc1"
#######################################################
#   srem    移除set集合里的指定元素
127.0.0.1:6379>  srem myset hello
(integer) 1
127.0.0.1:6379> SMEMBERS myset
1) "cc"
2) "cc1"
#######################################################
#   srandmember 随机抽选出几个元素
127.0.0.1:6379> SRANDMEMBER myset # 随机抽选一个元素
"cc"
127.0.0.1:6379> SRANDMEMBER myset
"cc1"
127.0.0.1:6379> SRANDMEMBER myset 2 # 随机抽选出两个元素
1) "cc"
2) "cc1"
#######################################################
#   spop    随机移除几个元素
127.0.0.1:6379> SMEMBERS myset
1) "cc"
2) "cc1"
127.0.0.1:6379> spop myset
"cc1"
127.0.0.1:6379> spop myset
"cc"
#######################################################
#   smove   将指定的值移动到另一个集合 
127.0.0.1:6379> SMEMBERS myset
1) "cc"
2) "world"
3) "hello"
127.0.0.1:6379> SADD myset2 set2
(integer) 1
127.0.0.1:6379> smove myset myset2 cc
(integer) 1
127.0.0.1:6379> SMEMBERS myset
1) "world"
2) "hello"
127.0.0.1:6379> SMEMBERS myset2
1) "cc"
2) "set2"
#######################################################
#   数字集合类（差集、并集、交集）
127.0.0.1:6379> sadd key1 a
(integer) 1
127.0.0.1:6379> sadd key1 b
(integer) 1
127.0.0.1:6379> sadd key1 c
(integer) 1
127.0.0.1:6379> sadd key2 c
(integer) 1
127.0.0.1:6379> sadd key2 d
(integer) 1
127.0.0.1:6379> sadd key2 e
(integer) 1
127.0.0.1:6379> SDIFF key1 key2 # 差集
1) "a"
2) "b"
127.0.0.1:6379> SINTER key1 key2    # 交集  共同好友实现
1) "c"
127.0.0.1:6379> SUNION key1 key2    # 并集
1) "a"
2) "b"
3) "c"
4) "e"
5) "d"
```
set是无序不重复集合，抽随机
### Zset
在set的基础上，增加了一个值，set k1 v1，zset k2 score v2
```bash
#######################################################
zadd    #   添加一个或多个值
zrange  #   遍历查询
127.0.0.1:6379> zadd myset 1 one
(integer) 1
127.0.0.1:6379> zadd myset 2 two 3 three
(integer) 2
127.0.0.1:6379> ZRANGE myset 0 -1
1) "one"
2) "two"
3) "three"
#######################################################
#   zrangebyscore   根据score排序
127.0.0.1:6379> zset salary 2000 a
(error) ERR unknown command `zset`, with args beginning with: `salary`, `2000`, `a`, 
127.0.0.1:6379> zadd salary 2000 a
(integer) 1
127.0.0.1:6379> zadd salary 5000 b
(integer) 1
127.0.0.1:6379> zadd salary 500 c
(integer) 1
127.0.0.1:6379> ZRANGEBYSCORE salary -inf +inf #显示全部用户，从小到大
1) "c"
2) "a"
3) "b"
127.0.0.1:6379> ZRANGEBYSCORE salary -inf +inf withscores   #显示全部用户并且附带成绩
1) "c"
2) "500"
3) "a"
4) "2000"
5) "b"
6) "5000"
127.0.0.1:6379> ZREVRANGE salary 0 -1 withscores    # 显示全部用户，从大到小
1) "b"
2) "5000"
3) "c"
4) "500"

#######################################################
#   zrem    移除指定元素
#   zcard   获取集合中的个数
127.0.0.1:6379> ZRANGEBYSCORE salary -inf +inf
1) "c"
2) "a"
3) "b"
127.0.0.1:6379> ZREM salary a
(integer) 1
127.0.0.1:6379> ZRANGEBYSCORE salary -inf +inf
1) "c"
2) "b"
127.0.0.1:6379> zcard salary
(integer) 2
#######################################################
#   zcount  获取指定区间的个数
127.0.0.1:6379> zadd myset 1 hello
(integer) 1
127.0.0.1:6379> zadd myset 2 world 3 c
(integer) 2
127.0.0.1:6379> ZCOUNT myset 1 3
(integer) 3

```
### Hash
```bash
#######################################################
#   hset set一个具体的value
#   hget get一个具体的value
#   hmset set多个具体的value
#   hmget get多个具体的value
#   hgetall get所有
#   hdel    删除hash指定key字段！对应的value值也就消失了
127.0.0.1:6379> hset myhash field1 c
(integer) 1
127.0.0.1:6379> HGET myhash field1
"c"
127.0.0.1:6379> HMSET myhash field1 hello field2 world
OK
127.0.0.1:6379> HMGET myhash field1 field2
1) "hello"
2) "world"
127.0.0.1:6379> HGETALL myhash
1) "field1"
2) "hello"
3) "field2"
4) "world"
127.0.0.1:6379> HGETALL myhash
1) "field2"
2) "world"
3) "field1"
4) "hello"
127.0.0.1:6379> HDEL myhash field1
(integer) 1
127.0.0.1:6379> HGETALL myhash
1) "field2"
2) "world"
#######################################################
#   hlen    获取hash表的字符字段
127.0.0.1:6379> hgetall myhash
1) "field2"
2) "world"
127.0.0.1:6379> hlen myhash
(integer) 1
#######################################################
#   hexists 判断hash中指定字段是否存在
127.0.0.1:6379> hgetall myhash
1) "field2"
2) "world"
127.0.0.1:6379> HEXISTS myhash field2
(integer) 1
127.0.0.1:6379> HEXISTS myhash field1
(integer) 0
#######################################################
#   hkeys   只获得所有的field
#   hvals   只获得所有的value
127.0.0.1:6379> HKEYS myhash
1) "field2"
127.0.0.1:6379> HVALS myhash
1) "world"
#######################################################
#   hincrby 步长    自增
127.0.0.1:6379> hset myhash field3 5
(integer) 1
127.0.0.1:6379> HINCRBY myhash field3 1
(integer) 6
127.0.0.1:6379> HINCRBY myhash field3 -1
(integer) 5
#######################################################
#   hsetnx  没有元素则创建，有元素失败
127.0.0.1:6379> HSETNX myhash field4 hello
(integer) 1
127.0.0.1:6379> HSETNX myhash field4 hello1
(integer) 0

```
map集合， key-map，本质和string类型有太大区别，还是一个简单的key-value
使用场景：
- hash变更数据，hash更适合对象的存储，string更适合字符串的存储

