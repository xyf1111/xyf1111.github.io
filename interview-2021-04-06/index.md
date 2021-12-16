# Interview 2021 04 06

<!--more-->
## 自我介绍
## 说一下MVC架构
## slice切片的概念
在Go中slice不仅是一个切片动作，还是一种数据结构。

slice结构都由三部分组成：容量，长度和指向底层数组某元素的指针。

虽然nil slice和空slice的长度和容量虽然都为0，输出结果也都为[]，且都不存储任何数据，但是它们是不同的。nil slice不会指向底层数组，而空slice会指向底层数组，只不过这个底层数组暂时是空数组
## 一个生产者多个消费者怎么使用channel
## 双重for循环怎么跳出
方法1：内层for循环赋值一个bool，break跳出，外层for判断bool，break跳出
方法2：使用goto+标签跳出多层循环（label标签）
方法3：使用break+标签跳出循环
## redis五种数据结构
string list set zset hash
## 经常使用redis的哪种数据结构
## Gin框架的了解
## map和slice分别怎么删除元素
map直接使用delete

slice取需要删除的元素其他切片元素
## go中interface的概念
接口是一种抽象类型，是多个方法声明的集合。在Go中只要目标类型实现了接口要求的所有方法，就说它实现了这个接口

使用type关键词定义接口
```go
type Duck interface {
    Eat()
}
```

接口赋值分为两种情况
1. 将对象实例赋值给接口
```go
var c Duck = &Cat{}
c.Eat()
```
将对象实例赋值给接口时，一定要传结构体指针，如果直接传结构体会报错
2. 将一个接口赋值给另一个接口
已经初始化的接口变量c1直接赋值给另一个接口变量c2，要求c2的方法集是c
1方法集的子集

具有0个方法的接口称为空接口，它的表示为interface{}。由于空接口有0个方法，所以所有类型都实现了空接口

类型断言
```go
x.(T)
```
其中x是接口类型的表达式，T是断言类型。作用是判断操作数的动态类型是否满足指定的断言类型。如果断言只返回一个值，类型错误报panic，如果返回两个值，类型错误另一个值为false。
## orm使用的什么
## mysql的存储引擎
## go读写锁的使用
读写锁sync.RWMutex，写锁Lock和Unlock，读锁RLock和RUnlock

读写锁可以让多个读同时进行，同时读取，但对于写操作是完全互斥的。也就是说，当一个goroutine进行写操作时，其他的goroutine既不能进行读操作，也不能进行写操作。
## redis分布式锁
设置锁setnx key value，设置锁的过期时间setex key second value。

当redis为集群，获取到多数节点的获取锁。

