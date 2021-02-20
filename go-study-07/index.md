# Go Study 07

<!--more-->
## 面向对象
- go语言仅支持封装，不支持继承和多态
- go语言没有class，只有struct
## 结构的创建
```go
type treeNode struct {
	value		int
	left, right	*treeNode
}
//自定义工厂函数
func createNode(value int) *treeNode {
    //返回局部变量地址
	return &treeNode{value: value}
}

func main() {
	var root treeNode
	fmt.Println(root)

	root = treeNode{value: 3}
	root.left = &treeNode{}
	root.right = &treeNode{5, nil, nil}
	root.right.left = new(treeNode)
	root.left.right = createNode(2)
}
```
- 不论地址还是结构本身，一律使用.来访问成员
- 使用自定义工厂函数(createNode)
- 注意返回了局部变量的地址
## 为结构定义方法
```go
func (node treeNode) print() {
	fmt.Println(node.value)
}
```
- 显示定义和命名方法接受者
## 使用指针作为方法接受者
```go
func (node *treeNode) setValue(value int) {
	node.value = value
}
```
- 只有使用指针才可以改变结构内容
- nil指针也可以调用方法
## 值接受者 vs 指针接受者
- 要改变内容必须使用指针接收者
- 结构过大也考虑使用指针接收者(因为值接收者使用时会拷贝一份，结构过大拷贝代价也大)
- 一致性：如有指针接收者，最好都使用指针接收者
- 很简单的不可变对象使用值接收者可以减轻GC负担(太多的指针会增加垃圾服务器GC的负担)
- **值接收者**是go语言特有
- 值/指针接收者均可接收值/指针
## 案例
### 结构体
Go的结构体是各个字段 字段类型的集合
```go
package main

import "fmt"

type person struct {
	name	string
	age		int
}

func main() {
	//使用这个语法创建一个新的结构体函数
	fmt.Println(person{"Bob", 20})
	//可以在初始化一个结构体元素时指定字段名
	fmt.Println(person{name:"Alice", age:18})
	//省略的字段将被初始化为零值
	fmt.Println(person{name:"Fred"})
	//&前缀生成一个结构体指针
	fmt.Println(&person{name:"Ann", age:40})

	//使用点来访问结构体字段
	s := person{name:"Sean", age:50}
	fmt.Println(s.name)
	fmt.Println(s.age)

	//也可以对结构体指针引用，指针会被自动解引用
	sp := &s
	fmt.Println(sp.age)

	//结构体是可变的
	sp.age = 51
	fmt.Println(s.age)
	s.age = 52
	fmt.Println(s.age)
}
```
### 方法
Go支持在结构体类型中定义方法
```go
package main

import "fmt"

type rect struct {
	wight	int
	height	int
}

//area方法有一个接收器类型rect来计算矩形面积
func (r rect) area() int {
	return r.height * r.wight
}

//可以为值或指针类型的接收器定义方法，这是个值类型接收器
//计算矩形周长
func (r rect) perim() int {
	return 2*r.wight + 2*r.height
}

//指针类型接收器可以改变实际值
func (r *rect) changeH(val int) {
	r.height = val
}

//值类型接收器改变拷贝值
func (r rect) changeW(val int) {
	r.wight = val
}

func main() {
	r := rect{10, 5}
	//调用上面为结构体定义的方法
	fmt.Println(r.area())
	fmt.Println(r.perim())
	r.changeH(20)
	r.changeW(15)
	fmt.Println(r)

	//Go自动处理方法调用时值和指针之间的转换
	//可以使用指针来调用方法避免在方法调用时产生一个拷贝或让方法能够改变接收的数据
	rp := &r
	rp.changeH(25)
	rp.changeW(30)
	fmt.Println(rp.area())
	fmt.Println(rp.perim())
}
```
### 值和指针
Go支持指针，允许在程序中通过**引用**传递值或数据结构
```go
package main

import "fmt"
//通过两个不同的函数来比较值和指针类型的不同
//zeroVal有一个int型参数，所以使用值传递
//zeroVal将从调用它的函数中获得一个ival形参的拷贝
func zeroVal(ival int)  {
	ival = 0
}
//zeroPtr和上面不同是*int，意味着它使用的是指针
//函数体内的*iptr接着解引用这个指针，从它内存地址得到这个地址当前值
//对一个解引用指针进行赋值会改变这个指针引用的真实地址的值
func zeroPtr(iptr *int)  {
	*iptr = 0
}

func main() {
	i := 1
	fmt.Println("initial: ", i)
	zeroVal(i)
	fmt.Println("zeroVal: ", i)
	//通过&i语法来获取i的内存地址
	zeroPtr(&i)
	//指针也是可以被打印的
	fmt.Println("zeroPtr: ", i)

	fmt.Println("point: ", &i)
}
```
