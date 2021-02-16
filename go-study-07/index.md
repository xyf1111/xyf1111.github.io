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
- 结构过大也考虑使用指针接收者
- 一致性：如有指针接收者，最好都使用指针接收者
- **值接收者**是go语言特有
- 值/指针接收者均可接收值/指针
