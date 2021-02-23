# Go Instance 04

<!--more-->
## goroutine和channel
### goroutine和channel协同工作
具体要求
1. 开启一个writeData协程，向管道intChan中写入50个整数
2. 开启一个readData协程，从管道intChan中读取writeData写入的数据
3. 注意writeData和readData操作的是同一个管道
4. 主线程需要等待writeData和readData协程都完成工作才能退出管道

思路分析
!["思路分析"](/images/instance04-01.png "思路分析")
```go
package main

import "fmt"

func writeData(intChan chan int)  {
	for i := 1; i <= 50; i++ {
		intChan <-i
		fmt.Println("write:", i)
	}
	close(intChan)
}

func readData(intChan chan int, exitChan chan bool)  {
	for {
		n, ok := <-intChan
		if !ok {
			break
		}
		fmt.Println("read:", n)
	}
	exitChan <- true
	close(exitChan)
}

func main() {
	intChan := make(chan int, 50)
	exitChan := make(chan bool, 1)

	go writeData(intChan)
	go readData(intChan, exitChan)

	for {
		_, ok := <-exitChan
		if !ok {
			break
		}
	}
}
```
读和写频率不一样也没有问题
```go
package main

import (
	"fmt"
	"time"
)

func writeData(intChan chan int)  {
	for i := 0; i < 50; i++ {
		intChan <- i
		fmt.Println("write:", i)
	}
	close(intChan)
}

func readData(intChan chan int, exitChan chan bool) {
	for {
		v, ok := <-intChan
		if !ok {
			break
		}
		time.Sleep(time.Second)
		fmt.Println("read:", v)
	}
	exitChan <- true
	close(exitChan)
}

func main() {
	intChan := make(chan int, 10)
	exitChan := make(chan bool, 1)

	go writeData(intChan)
	go readData(intChan, exitChan)
	for {
		if _, ok := <-exitChan; !ok{
			break
		}
	}
}
```
注意：如果只向管道写入数据而没有读取，就会造成堵塞而deadlock
