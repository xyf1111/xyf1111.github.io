# Go Instance 10

<!--more-->
## 原子计数器
Go中最主要的状态管理方法是通过通道之间的通信完成的。在工作池的例子中有遇到，但是还是有一些其他方式来管理状态。如何使用sync/atomic包在多个Go协程中进行原子计数
```go
package main

import (
	"fmt"
	"runtime"
	"sync/atomic"
	"time"
)
//原子计数器
func main() {
	//使用一个无符号整型(永远是正整数)来代表这个计数器
	var ops uint64 = 0
	//为了模拟并发更新，启动50个协程，对计数器每个1毫秒进行一次加一操作
	for i := 0; i < 50; i++ {
		go func() {
			for {
				//使用AddUint64来让计数器自动增加，使用&语法来抛出ops内存地址
				atomic.AddUint64(&ops, 1)
				//允许其他Go协程执行
				runtime.Gosched()
			}
		}()
	}

	//等待1秒,让Go协程有时间运行
	time.Sleep(time.Second)
	//在计数器还在被其他Go协程更新时,安全的使用
	//通过LoadUint将当前值拷贝到opsfinal
	//LoadUint64 需要的是内存地址
	opsfinal := atomic.LoadUint64(&ops)
	fmt.Println(opsfinal)
}
```
