# Go Interview 01

<!--more-->
## new和make的定义
```go
func new(Type) *Type
func make(t Type, size ...IntegerType) Type
```
其中Type代表一个数据类型
## 二者的区别
### 返回值
从定义中可以看出，new返回的是指向Type的指针，make直接返回的是Type类型值
### 入参
new只有一个Type参数，Type可以是任意数据类型。make可以有多个参数，但是只能是slice，map或者chan中的一种。对于不同类型，size参数说明如下：
- 对于slice,第一个size表示长度，第二个size表示容量，且容量不能小于长度。如果省略第二个size，默认容量等于长度。
- 对于map，会根据size大小分配资源，以足够存储size个元素。如果省略size，会默认分配一个小的起始size
- 对于chan，size表示缓冲区容量。如果省略size，channel为无缓冲channel
### 分配类型
- new：用来分配内存，主要用来分配值类型
- make：用来分配内存，主要用来分配引用类型
