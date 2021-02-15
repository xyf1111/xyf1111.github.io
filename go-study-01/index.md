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
