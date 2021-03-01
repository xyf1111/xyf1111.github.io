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
## 互斥锁
相较于简单的原子计数器，对于更复杂的情况，可以使用一个互斥锁在Go协程里安全的访问数据
```go
package main

import (
	"fmt"
	"math/rand"
	"runtime"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	//在这个例子中state是一个map
	var state = make(map[int]int)
	//这里的mutex将同步对mutex的访问
	var mutex = &sync.Mutex{}
	//为了比较基于互斥锁的处理方式和其他方式，ops将记录对state的操作次数
	var ops int64 = 0
	//运行100Go协程同时读取state
	for i := 0; i < 100; i++ {
		go func() {
			total := 0
			for {
				//每次循环读取，使用一个键进行访问
				//Lock()这个mutex来确保对state的独占访问，读取选定键的值
				//Unlock()这个state并使ops+1
				key := rand.Intn(5)
				//fmt.Println("key:", key)
				mutex.Lock()
				total += state[key]
				mutex.Unlock()
				atomic.AddInt64(&ops, 1)
				//为了确保这个Go协程不会在调度中饿死
				//每次操作后明确使用runtime.Gosched()进行释放
				//这个释放一般是自动处理的，例如每个通道操作后或者time.Sleep的阻塞调用后相似
				//但在这个例子中需要手动处理
				runtime.Gosched()
			}
		}()
	}

	//运行10个Go协程来模拟写操作，使用和读取相同模式
	for i := 0; i < 10; i++ {
		go func() {
			for {
				key := rand.Intn(5)
				value := rand.Intn(100)
				mutex.Lock()
				state[key] = value
				mutex.Unlock()
				atomic.AddInt64(&ops, 1)
				runtime.Gosched()
			}
		}()
	}
	//让这10个Go协程对state和mutex的操作运行1秒
	time.Sleep(time.Second)
	//获取并输出最终操作数
	opsFinal := atomic.LoadInt64(&ops)
	fmt.Println("ops:", opsFinal)
	//对state使用一个最终锁，显示是如何结束的
	mutex.Lock()
	fmt.Println(state)
	mutex.Unlock()
}
```
