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
## 案例
### 数组
在Go中数组是一个固定长度的数列。在Go中相对于数组而言，slice使用更多
```go
func arraySample()  {
	//创建了一个数组a来存放刚好5个int。元素类型和长度是数组类型的一部分
	//数组默认是零值
	var a [5]int
	fmt.Println(a)
	//使用array[index] = value语法来设置数组指定位置的值
	//使用array[index]来得到值
	a[4] = 100
	fmt.Println(a)
	fmt.Println(a[4])
	//使用内置函数len返回数组的长度
	fmt.Println("len: ", len(a))
	//使用这个语法一行内初始化一个数组
	b := [5]int{1, 2, 3, 4, 5}
	fmt.Println("dcl: ", b)
	//数组的存储类型是单一的，但是可以组合这些数据来构造多维的数据结构
	var twoD [2][3]int
	for i := 0; i < 2; i++ {
		for j := 0; j < 3; j++ {
			twoD[i][j] = i + j
		}
	}
	fmt.Println("2d: ", twoD)
}
```
### 切片
Slice是Go中一个关键的数据类型，是一个比数组更加强大的序列接口
```go
func sliceSample()  {
	//slice的类型仅由它所包含的元素决定(不像数组还需要元素个数)
	//创建一个长度非零的空slice，需要使用内建的方法 make
	//创建一个长度为3的string类型slice(初始化为零值)
	s := make([]string, 3)
	fmt.Println("emp: ", s)
	//通过slice[index] = value来设置值
	//slice[index]来获取值
	s[0] = "a"
	s[1] = "b"
	s[2] = "c"
	fmt.Println(s)
	fmt.Println(s[2])
	//通过len来获取slice长度
	fmt.Println("len: ", len(s))
	//作为基本操作的补充，slice支持比数组更多的操作
	//其中一个是内建的append，它返回一个包含一个或多个新值的slice
	//slice底层是由指针数组和len以及cap组成的，append不会改变slice地址
	s = append(s, "d")
	s = append(s, "e", "f")
	fmt.Println("append: ", s)
	//slice可以被copy
	//这里新建一个空的和s相同长度的slice c，并将s复制给c
	c := make([]string, len(s))
	copy(c, s)
	fmt.Println("copy c: ",c)
	//slice支持通过slice[low:high]语法进行切片处理
	l := s[2:5]
	fmt.Println("切片之后2-5slice: ", l)
	//这个slice从s[0]到(包含)s[5]
	l = s[:5]
	fmt.Println("从s[0]到s[5]: ", l)
	//这个slice从s[2]到s最后一个值
	l = s[2:]
	fmt.Println("从s[2]到slice最后一个值: ", l)
	//在一行代码中声明并初始化一个slice变量
	t := []string{"g", "h", "i"}
	fmt.Println("dcl: ", t)
	//slice可以组成多维数据结构
	twoD := make([][]int, 3)
	for i := 0; i < 3; i++ {
		innerlen := 4
		twoD[i] = make([]int, innerlen)
		for j := 0; j < innerlen; j++ {
			twoD[i][j] = i + j
		}
	}
	fmt.Println("2d: ", twoD)
}
```
### 关联数组
map是Go内置关联数据类型(在其他语言成为哈希或字典)
```go
func mapsSample()  {
	//要创建一个空的map，需要使用内建的map1 := make(map[key-type]value-type)
	m := make(map[string]int)
	fmt.Println(m)
	//使用经典的map1[key] = value语法来设置键值对
	m["k1"] = 1
	m["k2"] = 2
	fmt.Println("map: ", m)
	//使用map1[key]来获取一个键的值
	v1 := m["k1"]
	fmt.Println(v1)
	//当对一个map调用内建len，返回的是键值对数目
	fmt.Println("len: ", len(m))
	//内建的delete可以从一个map中移除键值对
	delete(m, "k2")
	fmt.Println("delete", m)
	//从一个map中取值时，可选第二个返回值是这个键是否存在于map中
	//这个可以用来消除键不存在或键有零值产生的歧义
	_, prs := m["k2"]
	fmt.Println("prs: ", prs)
	//一行申明和初始化一个map
	n := map[string]int{"foo": 1, "bar": 2, "zhang": 3}
	fmt.Println(n)
}
```
