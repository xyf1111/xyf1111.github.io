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

## With系列函数
此外，context包中还定义了四个With系列函数。
### WithCancel
WithCancel的函数如下：
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```
WithCancel返回带有新Done通道的父节点的副本。当调用返回的cancel函数或当关闭父上下文的Done通道时，将关闭返回上下文的Done通道，无论发生什么情况。

取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel。
```go
package main

import (
	"context"
	"fmt"
)

func gen(ctx context.Context) <- chan int {
	dst := make(chan int)
	n := 1
	go func() {
		for {
			select {
			case <-ctx.Done():
				//return结束该goroutine，防止泄露
				return
			case dst <- n:
				n++
			}
		}
	}()
	return dst
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	//当取完需要的整数后调用cancel
	defer cancel()
	for r := range gen(ctx) {
		fmt.Println(r)
		if r == 5 {
			break
		}
	}
}
```
上面的示例代码中，gen函数在单独的goroutine中生成整数并将它们发送到返回的通道。gen的调用者在使用生成的整数之后需要取消上下文，以免gen启动的内部goroutine发生泄漏。

### WithDeadline
WithDeadline的函数接口如下
```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```
返回父上下文的副本，并将d调整不迟于deadline。如果父上下文的d已经早于deadline，则WithDeadline(parent, d)在语义上等同于父上下文。当截止日过期时，当调用返回的cancel函数时，或者当父上下文的Done通道关闭时，返回上下文的Done通道将被关闭，以最先发生的情况为准。

取消上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	d := time.Now().Add(time.Millisecond * 500)
	ctx, cancel := context.WithDeadline(context.Background(), d)
	defer cancel()

	select {
	case <-time.After(time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```
上面的代码中分，定义了一个500毫秒之后过期的deadline，然后我们调用context.WithDeadline(context.Backgroud(), d)得到一个上下文(ctx)和一个取消函数(cancel)，然后使用一个select让主程序陷入等待：等待1秒后打印overslept退出或者等待ctx过期后退出。因为ctx500毫秒后就过期了，所以ctx.Done()会先接收到值，上面代码会打印ctx.Err()取消原因。

### WithTimeout
WithTimeout的函数接口如下
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```
WithTimeout返回WithDeadline(parent, time.Now().Add(timeout))

取消此上下文将释放与其相关的资源，因此代码应该在上下文中运行的操作完成后立即调用cancel，通常用于数据库或者网络连接的超时控制。具体示例如下：
```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

var wg8 sync.WaitGroup

func work8(ctx context.Context) {
	LOOP:
		for {
			fmt.Println("db connect...")
			//假设正常连接数据库需要10毫秒
			time.Sleep(time.Millisecond * 10)
			select {
			//50毫秒后自动调用
			case <-ctx.Done():
				break LOOP
			default:
			}
		}
		fmt.Println("work done")
		wg8.Done()
}

func main() {
	//设置一个50毫秒的超时
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*50)

	wg8.Add(1)
	go work8(ctx)
	time.Sleep(time.Second * 5)
	//通知子goroutine结束
	cancel()
	wg8.Wait()
	fmt.Println("over")
}
```

### WithValue
WithValue 函数能够将请求作用域的数据与Context对象建立联系。函数接口如下：
```go
func WithValue(parent Context, key, val interface{}) Context
```
WithValue返回父节点的副本，其中与key关联的值是val。

仅对API和进程传递请求域的数据使用上下文值，而不是使用它来传递可选参数给函数。

所提供的键必须是可比较的，并且不应该是string类型或任何其他内置类型，以避免使用上下文在包之间发生冲突。WithValue的用户应该为键定义自己的类型。为了避免在分配给interface{}时进行分配，上下文通常具有具体类型struct{}。或者，导出的上下文关键变量的静态类型应该是指针或接口。
```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

var wg9 sync.WaitGroup
type TraceCode string

func work9(ctx context.Context) {
	key := TraceCode("Trace_Code")
	//在子goroutine中获取TraceCode
	traceCode, ok := ctx.Value(key).(string)
	if !ok {
		fmt.Println("invalid trace code")
	}
	LOOP:
		for {
			fmt.Printf("worker, trace code: %s\n", traceCode)
			//假设正常连接数据库需要10毫秒
			time.Sleep(time.Millisecond * 10)
			select {
			//50毫秒后自动调用
			case <-ctx.Done():
				break LOOP
			default:
			}
		}
		fmt.Println("work done")
	wg9.Done()
}

func main() {
	//设置一个50毫秒的超时
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*50)
	//在系统入口设置trace code传递给后续启动的子goroutine实现日志数据聚合
	ctx = context.WithValue(ctx, TraceCode("Trace_Code"), "12345")
	wg9.Add(1)
	go work9(ctx)
	time.Sleep(time.Second * 5)
	//通知子goroutine结束
	cancel()
	wg9.Wait()
	fmt.Println("over")
}
```
## 使用Context的注意事项
- 推荐以参数的方式显示传递Context。
- 以Context作为参数的函数方法，应该把Context作为第一参数。
- 给一个函数方法传递Context的时候，不要传递nil，如果不知道传递什么，就使用context.TODO()
- Context的Value相关方法应该传递请求域的必要数据，不应该用于传递可选参数。
- Context是线程安全的，可以放心的在多个goroutine中传递。
