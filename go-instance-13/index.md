# Go Instance 13

<!--more-->
## Panic
panic意味着有些出乎意料的错误发生。通常用它表示程序正常运行中不该出现的，或者没有处理好的事务
```go
package main

import "os"

func main() {
	//这里直接panic出一个错误
	panic("a problem")

	//panic的一个基本用法就是在一个函数返回了错误值但是并不知道（或不想）处理时停止运行
	//这里是创建一个新文件时返回异常错误的panic用法
	_, e := os.Create("F://test.txt")
	if e != nil {
		panic(e)
	}
}
```
## Defer
defer用来确保一个函数调用在程序执行结束前执行。通常用来执行一些清理工作
```go
package main

import (
	"fmt"
	"os"
)

//本例为创建一个文件，对其进行写操作，结束后关闭
func main() {
	//创建文件，进行写操作，结束时关闭文件
	f := createFile("F:/test/test.txt")
	defer closeFile(f)
	writeFile(f)
}

func createFile(s string) *os.File {
	fmt.Println("creating")
	file, err := os.Create(s)
	if err != nil {
		panic(err)
	}
	return file
}

func closeFile(f *os.File) {
	fmt.Println("closing")
	f.Close()
}

func writeFile(f *os.File) {
	fmt.Println("writing")
	fmt.Fprintln(f, "data ok")
}
```
