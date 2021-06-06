# GoContextStudy

<!--more-->
## Go标准库Context
在Go http包的server中，每一个请求在都有一个对应的goroutine去处理。请求处理函数通常会启动额外的goroutine用来访问后端服务，比如数据库和RPC服务。用来处理一个请求的goroutine通常需要访问一些与请求特定的数据，比如终端用户的身份认证信息，验证相关的token，请求的截止时间。当一个请求被取消或超时时，所有用来处理该请求的goroutine都应该迅速退出，然后系统才能释放这些goroutine占用的资源。
## 为什么需要Context
### 基本示例
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup

// 初始的例子

func work() {
	for {
		fmt.Println("worker")
		time.Sleep(time.Second)
    }
    // 如何接收外部命令实现退出
	wg.Done()
}

func main() {
	wg.Add(1)
	go work()
    // 如何优雅的实现结束子goroutine
	wg.Wait()
	fmt.Println("over")
}
```
### 全局变量方式
```go
package main

import (
	"fmt"
	"sync"
	"time"
)
//全局变量方式存在的问题：
//1.使用全局变量在跨包调用时不容易统一
//2.如果在worker中再启动goroutine，就不太好控制了

var wg2 sync.WaitGroup
var exit bool

func work2() {
	for {
		fmt.Println("work")
		time.Sleep(time.Second)
		if exit {
			break
		}
	}
	wg2.Done()
}

func main() {
	wg2.Add(1)
	go work2()
	//sleep3秒避免程序过早退出
	time.Sleep(time.Second * 3)
	//修改全局变量实现子goroutine的退出
	exit = true
	wg2.Wait()
	fmt.Println("over")
}
```
### 通道方式
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup

//管道方式存在的问题
//1.使用全局变量在跨包调用时不容易实现规范和统一，需要维护一个共同的channel

func work3(exitChan chan struct{}) {
	LOOP:
		for {
			fmt.Println("work")
			time.Sleep(time.Second)
			select {
			//等待接收上级通知
			case <- exitChan:
				break LOOP
			default:
			}
		}
		 wg.Done()
}

func main() {
	wg.Add(1)
	exitChan := make(chan struct{})
	go work3(exitChan)
	//sleep3秒避免程序过早退出
	time.Sleep(time.Second * 3)
	//给子goroutine发送退出信号
	exitChan <- struct {}{}
	close(exitChan)
	wg.Wait()
	fmt.Println("over")
}
```
### 官方版的方案
```
gopackage main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

var wg4 sync.WaitGroup

func work4(ctx context.Context) {
	LOOP:
		for {
			fmt.Println("work4")
			time.Sleep(time.Second)
			select {
			case <- ctx.Done():
				break LOOP
			default:
			}
		}
		wg4.Done()
}

func main() {
	wg4.Add(1)
	ctx, cancel := context.WithCancel(context.Background())
	go work4(ctx)
	time.Sleep(time.Second*3)
	cancel()
	wg4.Wait()
	fmt.Println("over")
}
```
### 当子goroutine又开启另一个goroutine时，只需将ctx传入即可
```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

var wg5 sync.WaitGroup

func work5(ctx context.Context) {
	go work5_2(ctx)
	LOOP:
		for {
			fmt.Println("work5")
			time.Sleep(time.Second)
			select {
			//等待上级通知
			case <-ctx.Done():
				break LOOP
			default:
			}
		}
		wg5.Done()
}

func work5_2(ctx context.Context) {
	LOOP:
		for {
			fmt.Println("work5_2")
			time.Sleep(time.Second)
			select {
			//等待上级通知
			case <-ctx.Done():
				break LOOP
			default:
			}
		}
}

func main() {
	wg5.Add(1)
	ctx, cancel := context.WithCancel(context.Background())
	go work5(ctx)
	time.Sleep(time.Second * 3)
	//通知子goroutine结束
	cancel()
	wg5.Wait()
	fmt.Println("over")
}
```
## Context初识
Go1.7加入了一个新的标准库context，它定义了Context类型，专门用来简化对于处理单个请求的多个goroutine之间与请求域的数据，取消信号、截止时间等相关操作，这些操作可能涉及多个API调用。

对服务器传入的请求应该创建上下文，而对服务器的传出调用应该接受上下文，它们之间的函数调用链必须传递上下文，或者可以使用WithCancel、WithDeadline、WithTimeout或WithValue创建派生的上下文。当一个上下文被取消时，它派生的所有上下文也被取消。

## Context接口
context.Context是一个接口，该接口定义了四个需要实现的方法。具体接口如下：
```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```
其中：
- Deadline方法返回需要返回当前Context被取消的时间，也就是完成工作的时间
- Done方法返回需要一个Channel，这个Channel会在当前工作完成或者上下文被取消后关闭，多次调用Done方法会返回同一Channel
- Err方法会返回当前Context结束的原因，它只会在Done返回的Channel被关闭时才会返回非空的值
    - 如果当前的Context被取消就会返回Canceled错误
    - 如果当前的Context超时就会返回DeadlineExceeded错误
- Value方法会从Context中返回键对应的值，对于同一个上下文来说，多次调用Value并传入相同的Key会返回相同的结果，该方法仅用于传递跨API和进程间跟请求域的数据

### Background()和TODO()
Go内置两个函数：Background()和TODO()，这两个函数分别返回一个实现了Context接口的background和todo。我们代码中最开始都是以这两个内置的上下文对象作为最顶层的parent context，衍生出更多的子上下文对象。

Background()主要作用于main函数，初始化以及测试代码中，作为Context这个树结构最顶层的Context，也就是根Context。

TODO()，它目前还不知道具体的使用场景，如果我们不知道该使用什么Context的使用可以使用这个。

backgroud和todo本质上都是emptyCtx结构类型，是一个不可取消，没有设置截止时间，没有携带任何值的Context。
