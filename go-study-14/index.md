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
