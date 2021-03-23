# Go Instance 18

<!--more-->
## 随机数
Go的math/rand包提供了伪随机数生成器
```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	//rand.Intn返回一个随机整数n，0<=n<=100
	fmt.Print(rand.Intn(100),",")
	fmt.Print(rand.Intn(100))
	fmt.Println()

	//rand.Float64返回一个64位浮点数f，0.0<=f<=1.0
	fmt.Println(rand.Float64())

	//这个技巧可以用来生成其他范围的随机浮点数f，5.0<=f<=10
	fmt.Print((rand.Float64()*5)+5, ",")
	fmt.Print((rand.Float64()*5)+5)
	fmt.Println()

	//默认情况下，给定的种子是确定的，每次都会产生相同的随机数数字序列
	//要产生变化的序列，需要给定一个变化的种子，需要注意的是
	//如果是出于加密目的，需要使用随机数的话，请使用crypto/rand包，此方法不够安全
	s1 := rand.NewSource(time.Now().UnixNano())
	r1 := rand.New(s1)
	fmt.Println(r1)

	//调用上面rand.Source的函数和调用rand包中的函数是相同的
	fmt.Print(r1.Intn(100), ",")
	fmt.Print(r1.Intn(100))
	fmt.Println()

	//如果使用相同种子生成的随机数生成器，将会产生相同的随机数列
	s2 := rand.NewSource(42)
	r2 := rand.New(s2)
	fmt.Print(r2.Intn(100), ",")
	fmt.Print(r2.Intn(100))
	fmt.Println()
	s3 := rand.NewSource(42)
	r3 := rand.New(s3)
	fmt.Print(r3.Intn(100), ",")
	fmt.Print(r3.Intn(100))
	fmt.Println()
}
```
