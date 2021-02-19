# Go Study 01

<!--more-->
## 变量定义
### 使用var关键字
- var a,b,c bool
- var s1,s2 string = "hello", "world"
- 可以放在函数内，或直接放在包内
- 使用var()集中定义变量
### 让编译器自动决定类型
- var a,b,i,s1,s2 = true, false, 3, "hello", "world"
### 使用:=定义变量
- a,b,i,s1,s2 := true, false, 3, "hello", "world"
- 只能在函数内使用
## 内建变量类型
- bool, string
- (u)int, (u)int8, (u)int16, (u)int32, (u)int(64), uintptr(指针)
- byte, rune
- float32, float64, complex64, complex128 (complex是复数)
## 强制类型转换
- 类型转换是强制的
- var a,b int = 3, 4
- var int = math.Sqrt(a*a + b*b) (错误)
- var int = int(math.Sqrt(float64(a*a + b*b))) (正确)
## 常量定义
- const filename = "abc.txt"
- const 数值可以作为各种类型使用
- const a, b = 3, 4
- var c int = int(math.Sqrt(a*a + b*b))
## 使用常量定义枚举类型
- 普通枚举类型
```go
    const (
		cpp = 1
		java = 2
		python = 3
		golang = 4
	)
```
- 自增值枚举类型
```go
    const (
		cpp = iota
		java
		python
		golang
	)
```
## 变量定义要点回顾
- 变量类型写在变量名之后
- 编译器可推测变量类型
- 没有char，只有rune
- 原生支持复数类型

## 案例
### 值
Go拥有各值类型，包括字符串，整型，浮点型，布尔型等
```go
func value()  {
	//字符串通过+连接
	fmt.Println("go" + "lang")
	//浮点数和整数
	fmt.Println("1+1=", 1+1)
	fmt.Println("7.0/3.0=", 7.0/3.0)
	//布尔型和逻辑运算符
	fmt.Println(true && false)
	fmt.Println(true || false)
	fmt.Println(!true)
}
```
### 变量
在Go中，变量被显式声明，并被编译器所用来检查函数调用时的类型正确性
```go
func varible()  {
	//声明一个变量
	var a string = "initial"
	fmt.Println(a)
	//声明多个变量
	var b, c int = 1, 2
	fmt.Println(b, c)
	//Go 将自动推断已经初始化的变量类型
	var d = true
	fmt.Printf("%T %v\n", d, d)
	//声明变量且没有给出对应初始值，变量会被初始化为零值
	var e int
	fmt.Println(e)
	//:=语句是申明并初始化变量的简写
	f := "short"
	fmt.Println(f)
}
```
### 常量
Go支持字符、字符串、布尔和数值常量
```go
func constant()  {
	//const用于声明一个常量
	const s  = "constant"
	fmt.Println("s=", s)
	//const语句可以出现在任何var语句可以出现的地方
	const n = 500000000
	fmt.Println(n)
	//常数表达式可以执行任意精度的运算
	const d  = 3e20 / n
	fmt.Println("d = 3e20/n : ", d)
	//数值型常量是没有确定的类型，直到他们被给定一个类型，比如说一次显示的类型转化
	fmt.Println("int64(d) = ", int64(d))
	//当上下文需要的时候，一个数可以被给定一个类型，比如变量赋值或函数调用
	//举个栗子，这里的math.Sin()需要一个float64参数
	fmt.Println("math.Sin(n) = ", math.Sin(n))
}
```
