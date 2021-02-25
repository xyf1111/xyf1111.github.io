# Go Instance 07

<!--more-->
## 定时器
当需要在后面一个时刻运行Go代码，或者在某段时间内重复运行。Go内置的定时器和打点器让这些很容易实现
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	//定时器表示在未来某一时刻的独立事件
	//提供定时器需要的时间，定时器将提供一个用于通知的通道
	//这里设置定时器将等待2秒
	timer1 := time.NewTimer(time.Second * 2)

	//<-timer1.C 直到这个定时器的通道C明确的发送了定时器失效的值之前，一直阻塞
	<-timer1.C
	fmt.Println("Timer 1 expired")

	//如果需要的仅仅是等待，可以使用Sleep
	//定时器有用的原因之一是可以在定时器失效前取消定时器
	timer2 := time.NewTimer(time.Second * 2)

	go func() {
		<- timer2.C
		fmt.Println("Timer 2 expired")
	}()

	stop := timer2.Stop()
	if stop {
		fmt.Println("Timer 2 stop")
	}
}
```
第一个定时器将在程序开始后2秒失效，第二个在其还没失效前就被停止了
## 打点器
定时器是当你想要在未来某一时刻执行一次时使用，打点器是当你想要在固定的时间间隔重复执行准备的。这是一个打点器的例子，它将定时执行，直到我们将它停止
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	
	//打点器和定时器的机制有点相似：一个通道用来发送数据。
	// 这里我们在这个通道上使用内置的 range 来迭代值每隔500ms 发送一次的值。
	ticker := time.NewTicker(time.Millisecond * 500)
	
	go func() {
		for t := range ticker.C {
			fmt.Println("Tick at ", t)
		}
	}()
	
	//打点器可以和定时器一样被停止。一旦一个打点停止了，将不能再从它的通道中接收到值。
	// 我们将在运行后 1600ms停止这个打点器。
	time.Sleep(time.Millisecond * 1000)
	ticker.Stop()
	fmt.Println("Ticker stopped")
}
```
