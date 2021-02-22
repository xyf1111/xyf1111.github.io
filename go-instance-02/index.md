# Go Instance 02

<!--more-->
## 读取文件
### 案例1
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
### 案例2
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
## 写入文件
### 案例1
创建一个新文件写入5句hello
```go
package main

import (
	"bufio"
	"fmt"
	"os"
)
//创建一个新文件写入5句hello
func main() {
	//打开文件，可写可创建
	file, err := os.OpenFile("fileOperations/write.txt", os.O_WRONLY|os.O_CREATE, 0666)
	if err != nil{
		fmt.Printf("open file err: %v\n", err)
	}
	//函数退出关闭文件
	defer file.Close()
	//写入文件，使用带缓存的*writer
	writer := bufio.NewWriter(file)
	for i := 0; i < 5; i++ {
		writer.WriteString("hello\n")
	}
	//Flush将缓存的文件真正写入到文件中
	writer.Flush()
	fmt.Println("文件写入成功")
}
```
### 案例2
打开已有文件追加内容
```go
package main

import (
	"bufio"
	"fmt"
	"os"
)
//打开已有文件追加内容
func main() {
	//打开文件，文件可写可以追加
	file, err := os.OpenFile("fileOperations/write.txt", os.O_APPEND|os.O_WRONLY, 0666)
	if err != nil {
		fmt.Println("file is err", err)
	}
	//函数最后关闭文件
	defer file.Close()
	//写入文件，使用带缓存的*writer
	writer := bufio.NewWriter(file)
	for i := 0; i < 5; i++ {
		writer.WriteString("abc \r\n")
	}
	//Flush将缓存文件真正写入到文件中
	writer.Flush()
}
```
### 案例3
打开一个存在的文件读取内容并追加内容
```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
)
//打开一个存在的文件读取内容并追加内容
func main() {
	//打开文件，可读可写可追加
	file, err := os.OpenFile("fileOperations/write.txt", os.O_APPEND|os.O_RDWR, 0666)
	if err != nil {
		fmt.Println("file open err:", err)
	}
	//函数结束关闭文件
	defer file.Close()
	//读取文件
	reader := bufio.NewReader(file)
	for{
		s, err := reader.ReadString('\n')
		fmt.Print(s)
		if err == io.EOF {
			break
		}
	}
	//写入文件
	writer := bufio.NewWriter(file)
	for i := 0; i < 5; i++ {
		writer.WriteString("hello world\n")
	}
	writer.Flush()
}
```
### 案例4
将一个文件内容复制到另一个文件中
```go
package main

import (
	"fmt"
	"io/ioutil"
	"os"
)
//将一个文件内容复制到另一个文件中
func main() {
	os.OpenFile("fileOperations/writecopy.txt", os.O_CREATE|os.O_WRONLY, 0666)
	file, err := ioutil.ReadFile("fileOperations/write.txt")
	if err != nil {
		fmt.Println("file1 open err:", err)
	}
	err = ioutil.WriteFile("fileOperations/writecopy.txt", file, 0666)
	if err != nil {
		fmt.Println("file2 copy err", err)
	}
}

```
