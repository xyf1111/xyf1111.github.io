# Go Study 11

<!--more-->
## 函数与闭包
### 函数式编程 vs 函数指针
- 函数式一等公民：参数，变量，返回值都可以是函数
- 高阶函数
- 函数 -> 闭包
### “正统”函数式编程
- 不可变性：不能有状态，只有常量和函数
- 函数只能有一个参数
### go 语言闭包的应用
- 更为自然，不需要修饰如何访问自由变量
- 没有Lambda表达式，但是有匿名函数
!["闭包与函数"](/images/functional1.png "闭包与函数")
## 案例
### 基本函数
函数是Go的中心
```go
package main

import "fmt"

//这里是一个函数，接受两个int，并以int返回它们的和
func plus(a, b int) int {
	//Go需要明确的返回值
	return a+b
}

func main() {
	//通过name(args)来调用一个函数
	result := plus(1, 2)
	fmt.Println(result)
}
```
### 多返回值函数
Go内建多返回值支持。这个特性在Go语言中经常被用到，例如用来同时返回一个函数的结果和错误信息
```go
package main

import "fmt"

//(int, int)在这个函数中标志着这个函数返回2个int
func vals() (int, int) {
	return 3, 7
}

func main() {
	//通过多赋值操作来使用两个不同的返回值
	a, b := vals()
	fmt.Println(a)
	fmt.Println(b)

	//如果想返回一部分值，可以使用空白定义符_
	_, c := vals()
	fmt.Println(c)
}

```
### 可变参数函数
可变参数函数。可以用任意数量的参数调用。例如，fmt.Println是一个常见的变参函数
```go
package main

import "fmt"

//这个函数使用任意数目的int作为参数
func sum(nums ...int)  {
	fmt.Print(nums, " ")
	total := 0
	for _, num := range nums {
		total += num
	}
	fmt.Println(total)
}

func main() {
	//变参函数使用常规的调用方法，除了参数比较特殊
	sum(1, 2)
	sum(1, 2, 3)

	//如果slice已经有多个值，想作为变参使用，可以这样调用func(slice...)
	nums := []int{1, 2, 3, 4, 5}
	sum(nums...)
}
```
### 闭包
Go支持通过闭包来使用匿名函数。匿名函数在你想定义一个不需要命名的内联函数时很实用
```go
package main

import "fmt"

//这个intSeq函数返回另一个在intSeq函数体内定义的匿名函数
//这个函数的返回使用闭包的方式 隐藏 变量 i
func intSeq() func() int {
	i := 0
	return func() int {
		i += 1
		return i
	}
}

func main() {
	//调用nextInt函数，将返回值(也是一个函数)赋给nextInt
	//这个函数的值包含了自己的值i，在每次调用nextInt都会更新i
	nextInt := intSeq()

	//多次调用闭包，查看效果
	fmt.Println(nextInt())
	fmt.Println(nextInt())
	fmt.Println(nextInt())
	fmt.Println(nextInt())
	fmt.Println(nextInt())

	//更换函数，确定每个函数的i是独立的
	nextInts := intSeq()
	fmt.Println(nextInts())

}
```
### 递归函数
Go支持递归，这是一个经典的阶乘案例
```go
package main

import "fmt"

//fact函数在到达fact(0)前一直在调用自身
func fact(n int) int {
	if n == 0 {
		return 1
	}
	return n * fact(n-1)
}

func main() {
	fmt.Println(fact(5))
}

```
