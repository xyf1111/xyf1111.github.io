# Go Interview 07

<!--more-->
## Go互斥锁的两种实现
1.用Mutex实现
```go
package main

import (
	"fmt"
	"sync"
)

var num int
var mtx sync.Mutex
var wg sync.WaitGroup

func add() {
	//mutex锁住
	mtx.Lock()
	//mutex解锁
	defer mtx.Unlock()
	//wg计数器减一
	defer wg.Done()

	num += 1
}

func main() {
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go add()
	}

	//wg.wait代替了之前的time.sleep。wg里面相当于有个计数器，只有计数器没了才继续进行，否则就阻塞。
	wg.Wait()
	fmt.Println(num)
}
```
2. 使用chan实现
```go
package main

import (
	"fmt"
	"sync"
)

var num int

func add(h chan int, wg *sync.WaitGroup)  {
	defer wg.Done()
	h <- 1
	num += 1
	<-h
}

func main() {
	var wg = &sync.WaitGroup{}
	h := make(chan int, 1)
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go add(h, wg)
	}
	wg.Wait()
	fmt.Println(num)
}
```
