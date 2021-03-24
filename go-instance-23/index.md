# Go Instance 23

<!--more-->
## 信号
有时候我们希望Go能智能的处理Unix信号。例如，希望当服务器接收到一个SIGTERM信号时能自动关机，或者一个命令号工具在接收到一个SIGINT信号时停止处理输入信息。
```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	sigs := make(chan os.Signal, 1)
	done := make(chan bool, 1)

	//signal.Notify注册这个给定的通道用于接收特定的信号
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

	//开启一个Go协程执行阻塞信号接收的操作
	//当sign接收到一个值,将其打印,然后通知程序可以退出了
	go func() {
		sig := <-sigs
		fmt.Println()
		fmt.Println(sig)
		done <- true
	}()

	//程序在这等待,直到得到了期待的信号,然后退出
	fmt.Println("awaiting signal")
	<- done
	fmt.Println("exiting")
}
```
## 退出
使用os.Exit来立即进行带给定状态的退出
```go
package main

import (
	"fmt"
	"os"
)

func main() {
	//当使用os.Exit时defer将不会执行
	defer fmt.Println("!")

	//退出并且退出状态为3
	os.Exit(3)
}
```
