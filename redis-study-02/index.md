# Redis Study 02

<!--more-->
## 五大数据类型
### Stirng（字符串）
```bash
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
127.0.0.1:6379> strlen key1 # 获取字符串长度
(integer) 7
127.0.0.1:6379> append key1 ",cc"
(integer) 10
127.0.0.1:6379> strlen key1
(integer) 10
127.0.0.1:6379> get key1
"v1hello,cc"
# 步长 i++
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
# 对象
127.0.0.1:6379[1]> set user:1 {name:zhangsan,age:3} # 设置一个user:1对象，值为json字符来保存一个对象
OK
127.0.0.1:6379[1]> mset user:1:name zhangsan user:1:age 3 # 这里的key是一个巧妙的设计：user:{id}:{filed}，如此设计在redis中是完全ok的
OK
127.0.0.1:6379[1]> mget user:1:name user:1:age
1) "zhangsan"
2) "3"
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
127.0.0.1:6379[1]> LINDEX list 0 # 通过下标获取list中的某一个值
"two"
127.0.0.1:6379[1]> LINDEX list 1
"one"
```
### Set
### Zset
### Hash
## 三大特殊数据类型
### geospatial
### hyperloglog
### bitmaps
