# Go Instance 02

<!--more-->
## 案例1
读取文件的内容并显示在终端(带缓冲区的方式)
```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
)
//读取文件的内容并显示在终端(带缓冲区的方式)
func main() {
	//打开文件
	file, err := os.Open("fileOperations/abc.txt")
	if err != nil {
		fmt.Println("文件打开失败")
	}
	//函数退出时关闭文件
	defer file.Close()
	//输出文件
	fmt.Printf("file=%v \n", &file)
	//创建一个*Reader，带缓冲的，默认缓冲区大小为4096
	reader := bufio.NewReader(file)
	//循环读取文件内容
	for {
		//读到一个换行结束
		s, err := reader.ReadString('\n')
		//输出文件内容
		fmt.Print(s)
		//io.EOF表示文件末尾
		if err == io.EOF {
			fmt.Println()
			break
		}
	}
	fmt.Println("文件读取结束")
}
```
## 案例2
对于文件不太大的情况，可以使用ioutil一次将整个文件读到内存里
```go
package main

import (
	"fmt"
	"io/ioutil"
)
//对于文件不太大的情况，可以使用ioutil一次将整个文件读到内存里
func main() {
	//打开文件
	file, err := ioutil.ReadFile("fileOperations/abc.txt")
	if err != nil {
		fmt.Println("open file err:", err)
	}
	//把读取到的文件显示在终端
	//不需要显式的Open和Close文件，因为这两个操作已经被封装在ioutilFile函数内部
	fmt.Printf("file is %v\n", file)
	fmt.Printf("file is %v\n", string(file))
}
```
