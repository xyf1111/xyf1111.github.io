# Go Interview 06

<!--more-->
## go两个协程交替打印1-100的奇数偶数
```go
package main

import (
	"fmt"
	"time"
)

const Pool = 100

func goroutine1(msg chan int) {
	for i:=1; i<=Pool; i++ {
		msg <- i
		if i%2 == 1 {
			fmt.Println("goroutine1:", i)
		}
	}
}

func goroutine2(msg chan int) {
	for i := 1; i <= Pool; i++ {
		<- msg
		if i%2 == 0 {
			fmt.Println("goroutine2:", i)
		}
	}
}

func main() {
	msg := make(chan int)
	go goroutine1(msg)
	go goroutine2(msg)
	time.Sleep(time.Second)
}
```
