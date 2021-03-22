# Go Instance 15

<!--more-->
## 字符串格式化
Go在传统的printf中对字符串格式化提供了优异的支持。
```go
package main

import (
	"fmt"
	"os"
)

type point struct {
	x, y	int
}

func main() {
	//Go为常规Go值的格式设计了许多种打印方式。
	p := point{1, 2}
	fmt.Printf("%v\n", p)

	//如果值是一个结构体，%+v的格式化输出内容包括结构体的字段名
	fmt.Printf("%+v\n", p)

	//%#v形式则输出这个值的Go语法表示。例如值的运行源代码片段
	fmt.Printf("%#v\n", p)

	//需要打印值的类型，使用%T
	fmt.Printf("%T\n", p)

	//格式化布尔值，%t
	fmt.Printf("%t\n", true)

	//格式化整型的多种方式
	//使用%d进行标准的十进制格式化
	fmt.Printf("%d\n", 123)

	//使用%b进行二进制格式化
	fmt.Printf("%b\n", 14)

	//这个整数对应的字符
	fmt.Printf("%c\n", 33)

	//%x提供十六进制编码
	fmt.Printf("%x\n", 456)

	//对于浮点数同样有很多格式化选项
	//使用%f进行最基本的十进制格式化
	fmt.Printf("%f\n", 78.9)

	//%e和%E将浮点型格式化为科学计数法表示形式
	fmt.Printf("%e\n", 123400000.0)
	fmt.Printf("%E\n", 123400000.0)

	//使用%s进行基本的字符串输出
	fmt.Printf("%s\n", "\"string\"")

	//带有双引号的输出，使用%q
	fmt.Printf("%q\n", "\"string\"")

	//%x输出使用base-16编码的字符串，每个字节用两个字符表示
	fmt.Printf("%x\n", "hey this")

	//输出一个指针的值%p
	fmt.Printf("%p\n", &p)

	//当输出数字时，想要控制输出结果的宽度和精度
	//可以在%后面使用数字来控制输出宽度
	//默认结果使用右对齐并且通过空格来填充空白部分
	fmt.Printf("|%6d|%6d|\n", 12, 345)

	//也可以指定浮点数的输出宽度，同时也可以通过宽度，精度的语法来指定输出的精度
	fmt.Printf("|%6.2f|%6.2f|\n", 1.2, 3.45)

	//要左对齐%后面使用-
	fmt.Printf("|%-6.2f|%-6.2f|\n", 1.2, 3.45)

	//字符串输出时的宽度
	fmt.Printf("|%6s|%6s|\n", "foo", "b")

	//要左对齐%后面使用-
	fmt.Printf("|%-6s|%-6s|\n", "foo", "b")

	//Printf通过os.Stdout输出格式化的字符串
	//Sprintf格式化并返回一个字符串并不带任何输出
	s := fmt.Sprintf("a %s", "string")
	fmt.Println(s)

	//使用Fprint格式化并输出到io.Writers而不是os.Stdout
	fmt.Fprint(os.Stderr, "an %s\n", "error")
}
```
## 正则表达式
Go提供内置的正则表达式。
```go
package main

import (
	"bytes"
	"fmt"
	"regexp"
)

func main() {

	//测试一个字段是否符合一个表达式
	matched, _ := regexp.MatchString("p([a-z]+)ch", "peach")
	fmt.Println(matched)

	//上面直接使用字符串，但是对于一些其他正则任务
	//需要Compile一个优化的Regexp结构体
	r, _ := regexp.Compile("p([a-z]+)ch")

	//这个结构体有很多方法，这是类似前面的匹配测试
	fmt.Println(r.MatchString("peach"))

	//这是查找匹配字符串
	fmt.Println(r.FindString("peach punch"))

	//查找第一次匹配的字符串，返回的是匹配开始和结束的索引，而不是内容
	fmt.Println(r.FindStringIndex("peach punch"))

	//submatch返回完全匹配和局部匹配的字符串。例如这里会返回p([a-z]+)ch和([a-z]+)的信息
	fmt.Println(r.FindStringSubmatch("peach punch"))

	//返回完全匹配和局部匹配的索引位置
	fmt.Println(r.FindStringSubmatchIndex("peach punch"))

	//带All的这个函数返回所有的匹配项，而不仅仅是首次匹配项
	fmt.Println(r.FindAllString("peach punch pinch", -1))

	//返回所有的完全匹配和局部匹配的索引位置
	fmt.Println(r.FindAllStringSubmatchIndex("peach punch pinch", -1))

	//使用提供的正整数限制匹配个数
	fmt.Println(r.FindAllString("peach punch pinch", 2))

	//也可以提供[]byte参数并将String从函数名中去掉
	fmt.Println(r.Match([]byte("peach")))

	//创建正则表达式常量时，可以使用Compile的变体MustCompile
	//因为Compile返回两个值，不能用于常量
	r = regexp.MustCompile("p([a-z]+)ch")
	fmt.Println(r)

	//regexp包也可以用来替换部分字符串的其他值
	fmt.Println(r.ReplaceAllString("a peach", "<fruits>"))

	//Func变量允许传递匹配内容到一个给定的函数中
	in := []byte("a peach")
	out := r.ReplaceAllFunc(in, bytes.ToUpper)
	fmt.Println(string(out))
}
```
