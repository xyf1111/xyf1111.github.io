# Go Study 12

<!--more-->
## defer调用
- 确保调用在函数结束时发生
- 多defer相当于栈(先进后出)
- 参数在defer语句时计算 
## 何时使用defer调用
- Open/Close
- Lock/Unlock
- PrintHeader/PrintFooter
## 错误处理二
- 如何实现统一的错误处理逻辑
## panic
- 停止当前函数运行
- 一直向上返回，执行每一层的defer
- 如果没有遇见recover，程序退出
## recover
- 仅在defer调用中使用
- 获取panic的值
- 如果无法处理，可重新panic
## error vs panic
- 意料之中的：使用error。如：文件打不开
- 意料之外的：使用panic。如：数组越界
## 错误处理综合实例
- defer + panic + recover
- Type Assertion
- 函数式编程的应用
## 案例
### 错误处理
Go语言使用一个独立的明确的返回值来传递错误信息。这与使用异常的Java和Ruby以及在C语言中常见到的超重的单返回值/错误值相比，Go语言的处理方式能清楚知道那个函数返回了错误，并能像调用那些没有出错函数一样调用
```go
package main

import (
	"errors"
	"fmt"
)

//按照惯例，错误通常是最后一个返回值并且是error类型，一个内建的接口
func f1(arg int) (int, error) {
	if arg == 42 {
		//error.New 构造一个使用给定错误信息的基本error值
		return -1, errors.New("can't work with 42")
	}
	//返回错误值为nil，代表没有错误
	return arg+3, nil
}

//通过实现Error方法来自定义error是可以的
//这里使用自定义错误类型来表示上面的参数错误
type argError struct {
	arg		int
	prob	string
}

func (e *argError) Error() string {
	return fmt.Sprintf("%d-%s", e.arg, e.prob)
}

func f2(arg int) (int, error) {
	if arg == 42 {
		//这个例子中，使用&argError语法来建立一个新的结构体
		//提供arg和prob两个字段的值
		return -1, &argError{arg, "can't work with it"}
	}
	return arg+3, nil
}

func main() {
	//下面两个循环测试了各个返回错误的函数
	//注意在if行内的错误检查代码，在Go中是一个普遍的用法
	for _, i := range []int{7, 42}{
		if r, e := f1(i); e != nil {
			fmt.Println("f1 failed: ", e)
		} else {
			fmt.Println("f1 worked: ", r)
		}
	}

	for _, i := range []int{7, 42} {
		if r, e := f2(i); e != nil {
			fmt.Println("f2 failed: ", e)
		} else {
			fmt.Println("f2 worked: ", r)
		}
	}

	//如果想在程序中使用一个自定义错误类型中的数据
	//需要通过类型断言来得到这个错误类型的实例
	_, e := f2(42)
	if ae, ok := e.(*argError); ok {
		fmt.Println(ae.arg)
		fmt.Println(ae.prob)
	}
}
```
