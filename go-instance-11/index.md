# Go Instance 11

<!--more-->
## Go协程状态
在前面的例子中，我们用互斥锁进行了明确的锁定来让共享的state 跨多个 Go 协程同步访问。另一个选择是使用内置的 Go协程和通道的的同步特性来达到同样的效果。这个基于通道的方法和 Go 通过通信以及 每个 Go 协程间通过通讯来共享内存，确保每块数据有单独的 Go 协程所有的思路是一致的。
```go
package main

import (
	"fmt"
	"math/rand"
	"sync/atomic"
	"time"
)

//在这个例子中，state将被一个单独的协程拥有
//这就能保证数据在并行读取时不会混乱，为了对state进行读取或写入
//其他Go协程将发送一条数据到拥有的Go协程中，然后接受对应的回复
//结构体readOp和writeOp封装这些状态，并且是拥有Go协程响应的一个方式
type readOp struct {
	key		int
	resp	chan int
}

type writeOp struct {
	key	int
	value	int
	resp	chan bool
}

func main() {
	//计算执行操作的次数
	var ops int64 = 0
	//reads和writes通道分别将被其他Go协程用来发布读和写请求
	reads := make(chan *readOp)
	writes := make(chan *writeOp)

	//这个就是拥有state的协程。这个协程的state是私有的
	//这个Go协程反复响应到达的请求
	//先响应到达的请求，然后返回一个值到响应通道resp表示操作成功
	go func() {
		var state = make(map[int]int)
		for {
			select {
			case read := <- reads:
				read.resp <- state[read.key]
			case write := <- writes:
				state[write.key] = write.value
				write.resp <- true
			}
		}
	}()

	//启动100个协程通过reads通道发起对state所有者Go协程的读取请求
	//每个读取请求需要构建一个readOp，发送它到reads通道中，并通过给定的resp接收结果
	for i := 0; i < 100; i++ {
		go func() {
			for {
				read := & readOp{
					key: rand.Intn(5),
					resp: make(chan int),
				}
				reads <- read
				<- read.resp
				atomic.AddInt64(&ops, 1)
			}
		}()
	}

	//用相同的方法启动10个写操作
	for i := 0; i < 10; i++ {
		go func() {
			for {
				write := &writeOp{
					key: rand.Intn(5),
					value: rand.Intn(100),
					resp: make(chan bool),
				}
				writes <- write
				<- write.resp
				atomic.AddInt64(&ops, 1)
			}
		}()
	}

	//让Go协程先跑1秒
	time.Sleep(time.Second)

	//最后获取并显示ops值
	opsfinal := atomic.LoadInt64(&ops)
	fmt.Println(opsfinal)
}
```

