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
