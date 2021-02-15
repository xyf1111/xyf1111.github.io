# Go Study 04

<!--more-->
## 数组
```go
    var arr1 [5]int
	arr2 := [3]int{1,3,5}
	arr3 := [...]int{2,4,6,8,10}
	var grid [4][5]int
```
- 数量写在类型前
## 数组的遍历
长度遍历
```go
    for i := 0; i < len(arr3); i++ {
		fmt.Println(arr3[i])
	}
```
range只取index
```go
    for i := range arr3 {
		fmt.Println(arr3[i])
	}
```
range取index和value
```go
    maxi := -1
	maxnum := -1
	for i, v := range arr3{
		if v > maxnum {
			maxi = i
			maxnum = v
		}
	}
	fmt.Println(maxi, maxnum)
```
range只取value
```go
    sum := 0
	for _, v := range arr3 {
		sum += v
	}
```
- 可以通过_省略变量
- 不仅range，任何地方都可以通过_省略变量
- 如果只要i，可写成 for i := range numbers
## 为什么要用range
- 意义明确，美观
## 数组是值类型
- [10]int 和[20]int 是不同类型
- 调用func f(arr [10]int) 会**拷贝**数组
- 在go语言中一般不直接使用数组
## Slice(切片)
```go
    arr := [...]int{0, 1, 2, 3, 4, 5, 6, 7}
    s := arr[2:6]
```
- s的值为[2 3 4 5]
```go
    arr := [...]int{0, 1, 2, 3, 4, 5, 6, 7}
    s := arr[2:6]
    s[0] = 10
```
- Slice 本身没有数据，是对底层array的一个view
- arr 的值变为[0 1 10 3 4 5 6 7]
## Reslice
```go
    s := arr[2:6]
    s = s[:3]
    s = s[1:]
    s = arr[:]
```
## Slice 的实现
!["slice的实现"](/images/slice1.png "slice的实现")
## Slice 的扩展
```go
    arr := [...]int{0, 1, 2, 3, 4, 5, 6, 7}
    s1 := arr[2:6]
    s2 := arr[3:5]
```
- s1 的值为[2 3 4 5]，s2 的值为[5 6]
- slice 可以向后扩展，不可以向前扩展
- s[i] 不可以超越 len(s)，向后扩展不可以超越底层数组cap(s)
## 向Slice 添加元素
- 添加元素时如果超越cap，系统就会重新分配更大的底层数组
- 由于值传递的关系，必须接受append的返回值
- s = append(s, val)
## 创建Slice
### 创建一个前99个奇数
```go
    var s []int

	for i := 0; i < 100; i++ {
		s = append(s, 2*i+1)
	}
```
### 创建一个确定的切片
```go
    s1 := []int{2, 4, 6, 8}
```
### 创建一个len为16的切片
```go
    s2 := make([]int, 16)
```
### 创建一个len为10，cap为32的切片
```go
    s3 := make([]int, 10, 32)
```
## 复制切片
```go
    copy(s2, s1)
```
## 删除切片的元素
### 删除切片里的一个元素(删除s2的第四个元素)
```go
    s2 = append(s2[:3], s2[4:]...)
```
### 删除切片第一个元素
```go
    s2 = s2[1:]
```
### 删除切片最后一个元素
```go
    s2 = s2[:len(s2)-1]
```
## Map
```go
    m := map[string]string {
        "name": "cc",
        "course":   "golang",
        "site": "im",
        "quality":  "notbad"
    }
```
- map[K]V, map[K1]map[K2]V
## Map的操作
### Map的遍历
- 使用range遍历key，或者遍历key，value对
```go
    for k, v := range m {
		fmt.Println(k, v)
	}
```
- 不保证遍历顺序，如需顺序，需手动对key排序
- 使用len来获得元素的个数
### 创建：make(map[string]int)
```go
    //m2 == empty
	m2 := make(map[string]int)
	//m3 == nil
	var m3 map[string]int
```
### 获取元素：map[key]
```go
    name := m["name"]
```
### key不存在时，获得Value类型的初始值
```go
    name1 := m["name1"]
```
### 用value, ok := map[key]来判断是否存在key
```go
    if name1, ok := m["name1"]; ok {
		fmt.Println(name1)
	} else {
		fmt.Println("key does not exist")
	}
```
### 使用delete来删除一个元素
```go
    delete(m, "age")
```
## Map的key
- map使用哈希表，必须可以比较相等
- 除了slice，map，function的内建类型都可以作为key
- Struct类型不包含上述字段，也可以作为key
