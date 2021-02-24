# Go Study 14

<!--more-->
## goroutine
### 协程Coroutine
- 轻量级“线程”
- **非抢占式**多任务处理，由协程主动交出控制权
- 编译器/解释器/虚拟机层面的多任务
- 多个协程可能在一个或多个线程上运行
!["coroutine"](/images/coroutine1.png "coroutine")
### goroutine的定义
- 任何函数只需加上go就能送给调度器运行
- 不需要在定义时区分是否是异步函数
- 调度器会在合适的点进行切换
- 使用-race来检测数据访问冲突
!["goroutine"](/images/goroutine1.png "goroutine")
### goroutine可能的切换点
- I/O，select
- channel
- 等待锁
- 函数调用(有时)
- runtime.Gosched()
- 以上只是参考，不能保证切换，不能保证在其他地方不切换
## 案例
### 协程
Go协程在执行上来说是轻量级的线程
```go
package main

import "fmt"

func f(s string)  {
	for i := 0; i < 3; i++ {
		fmt.Println(s)
	}
}
//Go协程在执行上来说是轻量级的线程
func main() {
	//使用一般方式运行f函数
	f("direct")
	//使用go f(s)在一个Go协程中调用这个函数
	//这个新的Go协程会并行的执行这个函数的调用
	go f("gorouting")
	//为匿名函数启动一个Go协程
	go func(msg string) {
		for i := 0; i < 10; i++ {
			fmt.Println(msg)
		}
	}("going")
	//现在这两个Go协程在独立的Go协程中异步运行，所以需要等他们结束
	//Scanln代码，我们在程序退出前按下任意键结束
	var in string
	fmt.Scanln(&in)
	fmt.Println("done")
}
```
当运行这个程序时，将先看见阻塞式调用，然后是两个go协程交替输出，这种交替输出表示Go运行时是以异步方式运行协程的
