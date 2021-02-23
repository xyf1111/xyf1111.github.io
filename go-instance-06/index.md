# Go Instance 06

<!--more-->
## defer+recover解决panic导致程序崩溃
### 案例1
如果我们起了一个协程，但这个协程出现了panic，但我们没有捕获这个协程，就会造成程序的崩溃，这时可以在goroutine中使用recover来捕获panic，进行处理，这样主线程不会受到影响
```go
package main

import (
	"fmt"
	"time"
)

func sayhello() {
	for i := 0; i < 10; i++ {
		fmt.Println("hello")
		time.Sleep(time.Second)
	}
}

func test() {
	//使用defer+recover
	defer func() {
		//捕获test抛出的panic
		if err := recover(); err != nil {
			fmt.Println("test 发生错误：", err)
		}
	}()

	var myMap map[int]string
	myMap[0] = "golang"
}

func main() {
	go sayhello()
	go test()

	for i := 0; i < 10; i++ {
		fmt.Println("main() ok=", i)
		time.Sleep(time.Second)
	}
}
```
