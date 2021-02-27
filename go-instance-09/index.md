# Go Instance 09

<!--more-->
## 速率限制
速率限制(英) 是一个重要的控制服务资源利用和质量的途径。Go 通过 Go 协程、通道和打点器优美的支持了速率限制。
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	//基本速率限制
	//如果想限制接收请求处理，可以将这些请求发送到一个相同通道
	requests := make(chan int, 5)

	for i := 1; i <= 5; i++ {
		requests <- i
	}
	close(requests)

	//这个tick通道将每200毫秒接收一个值，这个是速率限制任务中的管理器
	tick := time.Tick(time.Millisecond * 200)

	//通过在每次请求前阻塞tick通道的一个接收，限制每隔200毫秒接收一个值
	for row := range requests {
		<-tick
		fmt.Println("request:", row, time.Now())
	}

	//有时候想临时进行速率限制，并且不影响整体速率控制可以使用通道缓冲来实现
	//burstyLimter通道用来进行3次临时的脉冲型速率限制
	burstyLimter := make(chan time.Time, 3)

	//将需要临时改变值传入
	for i := 0; i < 3; i++ {
		burstyLimter <- time.Now()
	}
	//每个200毫秒添加一个新值到通道内，直到达到3个限制
	go func() {
		for t := range time.Tick(time.Millisecond * 200) {
			burstyLimter <- t
		}
	}()

	//现在模拟5个接入请求
	//刚开始3个受临时的脉冲影响
	burstyRequests := make(chan int, 5)

	for i := 0; i < 5; i++ {
		burstyRequests <- i
	}
	close(burstyRequests)

	for row := range burstyRequests {
		<- burstyLimter
		fmt.Println("request2: ", row, time.Now())
	}
}
```
运行程序，将看到第一批请求意料之中的大约每 200ms 处理一次。第二批请求，我们直接连续处理了 3 次，这是由于这个“脉冲”速率控制，然后大约每 200ms 处理其余的 2 个。
