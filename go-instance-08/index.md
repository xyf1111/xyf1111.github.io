# Go Instance 08

<!--more-->
## 工作池
在这个例子中，将看到如何使用GO协程和通道实现一个工作池
```go
package main

import (
	"fmt"
	"time"
)
//这是将要在多个并发实例中支持的任务
//这些执行者将从jobs通道接收任务，并通过results发送对应结果
//让每个任务睡1秒，模拟耗时任务
func worker(i int, jobs <-chan int, results chan<- int)  {
	for j := range jobs{
		fmt.Printf("worker %d processing job %d \n", i, j )
		time.Sleep(time.Second)
		results <- j
	}
}

func main() {
	//使用worker工作池并收集结果，需要两个通道
	jobs := make(chan int, 100)
	results := make(chan int, 100)

	//这里启动三个worker，初始是阻塞的，因为还没有任务
	for i := 1; i <= 3; i++ {
		go worker(i, jobs, results)
	}

	//发送9个任务，然后close表示这些就是所有任务了
	for i := 1; i <= 9; i++ {
		jobs <- i
	}
	close(jobs)

	//收集所有任务的返回值
	for i := 1; i <= 9; i++ {
		<-results
	}
}
```
执行这个程序，显示9个任务被多个worker执行。整个程序处理所有任务仅执行了3秒，因为是3个worker并行的
