# Go Instance 21

<!--more-->
## 读文件
```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"io/ioutil"
	"os"
)

func check(e error) {
	if e != nil {
		panic(e)
	}
}

func main() {
	//大部分文件读取都是将内容读取到内存中
	dat, err := ioutil.ReadFile("f:/test/test.txt")
	check(err)
	fmt.Println(string(dat))

	//读取文件可以从os.Open打开文件获取一个os.File开始
	f, e := os.Open("f:/test/test1.txt")
	defer f.Close()
	check(e)

	//从文件开始位置读取字节。这里最多读取5个字节。
	b1 := make([]byte, 5)
	n1, err1 := f.Read(b1)
	check(err1)
	fmt.Println(n1, string(b1))
	fmt.Printf("%d bytes: %s\n", n1, string(b1))

	//也可以Seek到一个文件已知的位置并从这个位置开始读取
	o2, err := f.Seek(6, 0)
	check(err)
	b2 := make([]byte, 2)
	n2, err := f.Read(b2)
	check(err)
	fmt.Printf("%d bytes @ %d: %s\n", n2, o2, string(b2))

	//io包提供了可以帮助文件读取的函数
	//可以使用ReadAtLeast得到一个更健壮的实现
	o3, err := f.Seek(6, 0)
	check(err)
	b3 := make([]byte, 2)
	n3, err := io.ReadAtLeast(f, b3, 2)
	check(err)
	fmt.Printf("%d bytes @ %d: %s\n", n3, o3, string(b3))

	//没有内置的回转支持，但是可以使用Seek(0,0)实现
	_, err = f.Seek(0, 0)
	check(err)

	//bufio包实现了带缓冲的读取
	//这不仅对有很多的小读取操作的性能有所提升，还提供了很多附加的读取函数
	r4 := bufio.NewReader(f)
	b4, err := r4.Peek(5)
	check(err)
	fmt.Printf("5 bytes: %s\n", string(b4))
}
```
## 写文件
```go
package main

import (
	"bufio"
	"fmt"
	"io/ioutil"
	"os"
)

//和read处于同一个包，不能有相同的方法名
//func check(err error)  {
//	if err != nil {
//		panic(err)
//	}
//}

func main() {
	//如何写入一个字符串到一个文件里
	d1 := "hello\ngo\n"
	err := ioutil.WriteFile("f:/test/test2.txt", []byte(d1), 0644)
	check(err)

	//更细粒度的写入，先打开一个文件
	f1, err := os.Create("f:/test/test3.txt")
	check(err)
	defer f1.Close()

	//写入要写的字节切片
	d2 := []byte{115, 111, 109, 101, 10}
	n, err := f1.Write(d2)
	check(err)
	fmt.Println(n)

	//WriteString也是可用的
	n3, err := f1.WriteString("writes\n")
	check(err)
	fmt.Println(n3)

	//调用sync将缓冲区的信息写入磁盘
	f1.Sync()

	//bufio提供带缓冲的写入器
	w := bufio.NewWriter(f1)
	n4, err := w.WriteString("buffered\n")
	check(err)
	fmt.Println(n4)

	//使用Flush确保所有缓存操作以写入磁盘
	w.Flush()
}
```
