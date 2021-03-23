# Go Instance 22

<!--more-->
## 行过滤器
一个行过滤器在读取标准输入流的输入，处理该输入，然后将得到一些的结果输出到标准输出的程序中是常见的一个功能。grep和sed是常见的行过滤器。
```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

//将所有的输入文字转化为大写的
func main() {
	//对os.Stdin使用一个带缓冲的Scanner
	//可以直接使用方便的Scan方法来直接读取一行
	//每次调用该方法可以让scanner读取下一行
	scanner := bufio.NewScanner(os.Stdin)

	//Text返回当前的token，现在是输入下一行
	if scanner.Scan() {
		str := strings.ToUpper(scanner.Text())
		//输出大写的行
		fmt.Println(str)
	}

	//检查Scan的错误，文件结束符是可以接收的，并且不会被Scan当做一个错误
	if err := scanner.Err(); err != nil {
		fmt.Fprintf(os.Stderr, "error:", err)
		os.Exit(1)
	}
}
```
## 命令行标志
命令行标志是命令行程序指定选项的常用方式
```go
package main

import (
	"flag"
	"fmt"
)

func main() {
	//基本的标记只支持字符串，整数和布尔值
	//声明一个默认值为foo的字符串标志word并带有一个简短的描述
	//这个flag.string返回的是一个指针
	wordPtr := flag.String("word", "foo", "a string")

	//使用和word标志相同的方式声明numb和fork标志
	numbPtr := flag.Int("numb", 42, "an int")
	boolPtr := flag.Bool("fork", false, "a bool")

	//用程序中已有的标志来声明一个标志也是可以的。注意在声明中使用参数的指针
	var svar string
	flag.StringVar(&svar, "svar", "bar", "a string var")

	//所有标志都声明完成后，调用flag.Parse()来执行命令解析
	flag.Parse()

	//输出解析的选项以及后面的位置参数
	fmt.Println("word:", *wordPtr)
	fmt.Println("numB:", *numbPtr)
	fmt.Println("fork:", *boolPtr)
	fmt.Println("svar:", svar)
	fmt.Println("tail:", flag.Args())
}
```
