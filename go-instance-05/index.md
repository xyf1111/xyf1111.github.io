# Go Instance 05

<!--mores-->
## 使用select解决从管道取数据堵塞问题
使用select解决从管道取数据堵塞的问题，语法如下：
!["select语法"](/images/instance05-01.png "select语法")
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	//定义一个管道，可以放10个int类型数据
	intChan := make(chan int, 10)
	for i := 0; i < 10; i++ {
		intChan <- i
	}
	//定义一个管道，可以放5个string类型数据
	stringChan := make(chan string, 5)
	for i := 0; i < 5; i++ {
		stringChan <- "hello" + fmt.Sprintf("%d", i)
	}
	//传统的方法遍历管道时，如果不关闭会阻塞而导致deadlock

	//实际开发中，不好确定什么时候关闭通道
	//这时可以使用select方式解决
	//label
	for {
		select {
		//管道不关闭不会deadlock，会自动到下一个case匹配
		case v := <-intChan:
			fmt.Printf("从intChan读取的数据%d\n", v)
			time.Sleep(time.Second)
		case v := <-stringChan:
			fmt.Printf("从stringChan读取的数据%v\n", v)
			time.Sleep(time.Second)
		default:
			fmt.Printf("取不到数据\n")
			time.Sleep(time.Second)
			return
		//break label
		}
	}
}

```
