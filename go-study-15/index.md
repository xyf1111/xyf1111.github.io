# Go Study 15

<!--more-->
## channel
### channel
### buffered channel(带缓冲区的channel)
### range(关闭channel)
### 理论基础Communication Sequential Process(CSP)
## 案例
### channel(通道)
通道是连接多个Go协程的管道，可以从一个Go协程将值发送到通道，然后在别的Go协程中接收
```go
package main

import (
	"fmt"
)

func channel1() {
	//使用make(chan val-type)创建一个新的通道
	//通道类型就是传递值的类型
	message := make(chan string)

	//使用 channel <- 语法发送一个新的值到通道中
	go func() {
		message <- "ping"
	}()

	fmt.Println(message)

	//使用 <- channel 语法从通道中获取一个值
	msg := <- message

	fmt.Println(msg)
}

func main() {
	channel1()
}
```
运行程序时，通过通道，消息“ping”成功的从一个Go协程传到另一个。默认发送和接收操作是阻塞的，直到发送方和接收方都准备完毕。这个特性允许不使用任何其他的同步操作，可以在程序结尾等待消息“ping”
### 通道缓冲
默认通道是无缓冲的，这意味着只有在对应的接收(<-chan)通道准备好接收时才允许发送(chan<-)。可缓冲通道允许在没有接收方的情况下，缓存限定数量的值
```go
package main

import (
	"fmt"
)

func channel2()  {
	//这里make了一个通道，最多允许缓存2个值
	message := make(chan string, 2)

	//因为这个通道有缓冲区，即使没有一个对应的并发接收方，也可以继续发送
	message <- "buffered"
	message <- "channel"
	//接收时，也可以接收两个值
	fmt.Println(<-message)
	fmt.Println(<-message)
}

func main() {
	channel2()
}
```
### 通道同步
可以使用通道来同步Go协程间的执行状态。这里使用阻塞的接收方式来等待一个Go协程的运行结束
```go
package main

import (
	"fmt"
	"time"
)

func channel3() {
	//创建一个缓存为1的bool类型通道
	done := make(chan bool, 1)
	//运行一个worker Go协程，并给予用于通知的通道
	go worker(done)
	//程序将在接收到通道中worker发出的通知前一直阻塞
	<-done
}

//这是一个我们将要在 Go 协程中运行的函数。
// done 通道将被用于通知其他 Go 协程这个函数已经工作完毕
func worker(done chan bool)  {
	fmt.Println("working...")
	time.Sleep(time.Second)
	fmt.Println("done")
	//发送一个值来通知完工啦。
	done <- true
}

func main() {
	channel3()
}
```
如果将done <- true这行代码从程序中移除，程序甚至会在worker还没开始运行就结束
### 通道方向
当使用通道作为函数参数时，可以指定通道是不是只用来接收或发送值。这个特性提升了程序的类型安全性
```go
package main

import (
	"fmt"
)

func channel4() {
	pings := make(chan string, 1)
	pongs := make(chan string, 1)
	ping(pings, "message")
	pong(pings, pongs)
	fmt.Println(<-pongs)
}

//ping函数定义一个只允许发送数据的通道
//尝试使用这个函数来接收数据会得到一个编译时错误
func ping(pings chan<- string, str string) {
	pings <- str
}
//pong函数允许通道pings接收函数，通道pongs发送函数
func pong(pings <-chan string, pongs chan<- string) {
	msg := <-pings
	pongs <- msg
}

func main() {
	channel4()
}
```
### 通道选择器
Go的通道选择器可以同时等待多个通道操作。Go协程和通道以及选择器的结合是Go的一个强大特性
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	//创建2个通道，从这两个选择
	c1 := make(chan string, 1)
	c2 := make(chan string, 1)

	//c1通道在1秒后接收值，这个模拟Go协程中阻塞的RPC操作
	go func() {
		time.Sleep(time.Second)
		c1 <- "one"
	}()

	//c2通道在2秒后接收值
	go func() {
		time.Sleep(time.Second * 2)
		c2 <- "two"
	}()

	//使用select关键字来同时等待这两个值，并打印各自的值
	for i :=0; i < 2; i++ {
		select {
		case msg := <-c1:
			fmt.Println(msg)
		case msg := <-c2:
			fmt.Println(msg)
		}
	}
}
```
注意两个协程并发运行，程序总共仅运行2秒左右
### 超时处理
超时对于一个连接外部资源，或者其他一些需要花费执行时间操作的程序而言是很重要的。得益于通道和select，在Go中实现超时操作是简洁而优雅的
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	//在这个例子中，执行一个外部调用，在2秒后通过通道c1返回执行结果
	c1 := make(chan string, 1)
	go func() {
		time.Sleep(time.Second * 2)
		c1 <- "result 1"
	}()

	//使用select实现超时操作
	//result := <-c1等待结果，<-Time.After等待超时1秒后发送的值
	//由于select默认处理第一个已准备好的接收操作，如果这个操作超过允许的1秒的话，将会执行超时case
	select {
	case result := <- c1:
		fmt.Println(result)
	case <-time.After(time.Second * 1):
		fmt.Println("result1 超时")
	}

	//执行一个外部调用，2秒后通过通道c2返回执行结果
	c2 := make(chan string, 1)
	go func() {
		time.Sleep(time.Second * 2)
		c2 <- "result 2"
	}()

	//将超时延长至3秒，如果操作小于3秒将执行result输出case
	select {
	case result := <-c2:
		fmt.Println(result)
	case <-time.After(time.Second * 3):
		fmt.Println("result2超时")
	}
}
```
运行程序，首先显示运行超时的操作，然后是成功接收的。使用select超时方式需要使用通道传递结果。这对于一般情况是个好方式，因为其他重要的Go特性是基于通道和select的
### 非阻塞式通道操作
常规的通过通道发送和接收数据是阻塞的。然而，可以使用带default子句select来实现非阻塞的发送、接收，甚至是非阻塞的多路select
```go
package main

import "fmt"
//这是一个非阻塞接收的例子
func main() {
	message := make(chan string, 1)
	signal := make(chan string, 1)

	//如果message有值存在，select将值传到msg中
	//如果不存在直接执行default
	select {
	case msg := <-message:
		fmt.Println("received message:", msg)
	default:
		fmt.Println("no message receives")
	}

	//这个非阻塞方法，如果msg成功向message传值，执行case
	//否则执行default
	msg := "hi"
	select {
	case message <- msg:
		fmt.Println("sent message:", msg)
	default:
		fmt.Println("no message sent")
	}

	signal <- "world"
	//可以在default前使用多个case子句实现一个多路的非阻塞的选择器
	//这里尝试在message和signal上同时使用非阻塞的接收操作
	//如果case1和case2都满足，只执行case1
	select {
	case msg := <-message:
		fmt.Println("receives message:", msg)
	case sign := <-signal:
		fmt.Println("receives signal:", sign)
	default:
		fmt.Println("no activity")
	}
}
```
### 通道的关闭
关闭一个通道意味着不能再向这个通道发送值了。这个特性可以用来给这个通道接收方传达工作已经完成的消息
```go
package main

import "fmt"

func main() {
	//在这个例子中，将使用一个jobs通道来传递main()中Go协程任务执行的结果信息到一个工作Go协程中
	//当没有多余任务给这个工作Go协程时，可以close这个jobs通道
	job := make(chan int, 5)
	done := make(chan bool, 1)

	//这是工作Go协程，使用j, more := <-job循环从job接收数据
	//在接收的这个特殊的二值形式的值中，如果job已经关闭，并且通道中的所有值都已经接收完毕
	//那么more的值将是false。当完成所有任务时，将使用这个特性通过done通道去进行通知
	go func() {
		for {
			j, more := <-job
			if more {
				fmt.Println("received job:", j)
			} else {
				fmt.Println("received all jobs")
				done <- true
				return
			}
		}
	}()

	//这里使用job发送3个任务到工作函数中，然后关闭job
	for i := 1; i <= 3; i++ {
		job <- i
		fmt.Println("sent job:", i)
	}

	close(job)
	fmt.Println("sent all job")

	//使用通道同步方法等待任务结束
	<-done
}
```
### 通道遍历
可以使用range语法来遍历从通道中取得的值。一个非空通道是可以关闭的，但是通道中的值仍然可以被接收到
```go
package main

import "fmt"

func main() {
	//遍历在queue通道中的两个值
	queue := make(chan string, 2)

	queue <- "one"
	queue <- "two"

	close(queue)

	//这个range迭代从queue中得到的每个值
	//因为close了通道，这个通道会在接收完第二个值后结束
	//如果没有close，那么这个循环将继续阻塞执行，等待接收第三个值
	for elem := range queue {
		fmt.Println(elem)
	}
}
```
