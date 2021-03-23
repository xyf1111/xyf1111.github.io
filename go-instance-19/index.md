# Go Instance 19

<!--more-->
## 数字解析
从字符串中解析数字
```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	//使用Parse.Float解析浮点数，64表示解析的数的位数
	f, _ := strconv.ParseFloat("1.234", 64)
	fmt.Println(f)

	//在使用ParseInt解析整数时，base参数0表示自动推断字符串所表示的数字进制
	//64表示返回的整型数是以64存储的
	i, _ := strconv.ParseInt("123", 0, 64)
	fmt.Println(i)

	//ParseInt会自动识别出十六进制数
	d, _ := strconv.ParseInt("0x1c8", 0, 64)
	fmt.Println(d)

	//ParseUint也是可用的
	u, _ := strconv.ParseUint("789", 0, 64)
	fmt.Println(u)

	//Atoi是一个基础的十进制整型数转换函数
	a, _ := strconv.Atoi("135")
	fmt.Println(a)

	//在输入错误时，会报错
	_, e := strconv.Atoi("wat")
	fmt.Println(e)
}
```
## URL解析
URL提供了一个同一资源定位方式
```go
package main

import (
	"fmt"
	"net/url"
	"strings"
)

func main() {
	//这个URL实例，包含一个Scheme，认证信息，主机名，端口，路径，查询参数和片段
	s := "postgres://user:pass@host.com:5432/path?k=v#f"

	//解析URL，确保解析没错
	u, e := url.Parse(s)
	if e != nil {
		panic(e)
	}

	//直接访问Scheme
	fmt.Println(u.Scheme)

	//User包含了所有的认证信息，调用Username和Password来获取独立值
	fmt.Println(u.User)
	fmt.Println(u.User.Username())
	p, _ := u.User.Password()
	fmt.Println(p)

	//Host同时包含主机名和端口信息，如果端口号存在的话，使用strings.Split从Host中手动提取
	fmt.Println(u.Host)
	h := strings.Split(u.Host, ":")
	fmt.Println(h[0])
	fmt.Println(h[1])

	//提取路径和查询片段信息
	fmt.Println(u.Path)
	fmt.Println(u.Fragment)

	//要得到字符串中的k=v这种格式的查询参数，可以使用RawQuery
	//也可以将查询参数解析为一个map。已解析的查询参数 map 以查询字符串为键，
	//对应值字符串切片为值，所以如何只想得到一个键对应的第一个值，将索引位置设置为 [0] 就行了。
	fmt.Println(u.RawQuery)
	m, _ := url.ParseQuery(u.RawQuery)
	fmt.Println(m)
	fmt.Println(m["k"][0])
}
```
