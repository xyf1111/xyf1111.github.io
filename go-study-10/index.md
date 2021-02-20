# Go Study 10

<!--more-->
## 接口
- 接口由使用者定义
## duck typing
- “像鸭子走路，像鸭子叫(长得像鸭子)，那么就是鸭子”
- 描述事物的外部行为而非内部结构
- 严格来说go属于结构化类型系统，类似duck typing
## 接口变量里有什么
!["接口变量1"](/images/interface1.png "接口变量1")
!["接口变量2"](/images/interface2.png "接口变量2")
- 接口变量自带指针
- 接口变量同样采用值传递，几乎不需要使用接口的指针
- 指针接收者实现只能以指针方式使用；值接收者都可以
## 查看接口变量
- 表示任何变量：interface{}
- Type Assertion
- Type Switch
## 特殊接口
- Stringer
- Reader/Writer
## 案例
### 接口
接口：Go语言中组织和命名相关的方法集合的机制。接口是方法特征的命名集合
```go
package main

import (
	"fmt"
	"math"
)

//这里是一个几何体的基本接口
type geometry interface {
	area()	float64
	perim()	float64
}

//这个例子中，将让rect和circle实现这个接口
type rect struct {
	width	float64
	height	float64
}

type circle struct {
	radius	float64
}

//在Go中实现接口，需要实现这个接口的所有方法
//rect实现接口
func (r rect) area() float64 {
	return r.width * r.height
}

func (r rect) perim() float64 {
	return 2*r.width + 2*r.height
}
//circle实现接口
func (c circle) area() float64 {
	return math.Pi * c.radius * c.radius
}

func (c circle) perim() float64 {
	return math.Pi * c.radius * 2
}
//如果有一个变量是接口类型，可以调用这个被命名接口的方法
//这里有一个通用的measure函数，利用特性，可以用在任何geometry(几何学)上
func measure(g geometry)  {
	fmt.Println(g)
	fmt.Println(g.area())
	fmt.Println(g.perim())
}

func main() {
	r := rect{width: 3, height: 4}
	c := circle{radius: 5}

	//结构体类型circle和rect都实现了geometry接口
	//所以可以使用他们的实例作为measure的参数
	measure(r)
	measure(c)
}
```
