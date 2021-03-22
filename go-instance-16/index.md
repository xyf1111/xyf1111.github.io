# Go Instance 16

<!--more-->
## JSON
Go提供内置的JSON编解码支持，包括内置或自定义类型与JSON数据之间的转化
```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
)

type Response1 struct {
	Page	int
	Fruits	[]string
}

type Response2 struct {
	Page	int `json:"page"`
	Fruits	[]string `json:"fruits"`
}

func main() {
	//基本数据类型到JSON字符串的编码过程
	bolB, _ := json.Marshal(true)
	fmt.Println(string(bolB))
	intB, _ := json.Marshal(1)
	fmt.Println(string(intB))
	fltB, _ := json.Marshal(3.14)
	fmt.Println(string(fltB))
	strB, _ := json.Marshal("gopher")
	fmt.Println(string(strB))

	//切片和map编码成JSON数组和对象
	slcD := []string{"apple", "peach", "pear"}
	slcB, _ := json.Marshal(slcD)
	fmt.Println(string(slcB))
	mapD := map[string]int{"apple": 5, "lettuce": 7}
	mapB, _ := json.Marshal(mapD)
	fmt.Println(string(mapB))

	//JSON包可以自动编码自定义类型
	//编码仅输出可导出的字段，并且默认使用他们的名字作为JSON数据的键
	res1D := &Response1{
		Page:1,
		Fruits:[]string{"apple", "peach", "pear"},
	}
	res1B, _ := json.Marshal(res1D)
	fmt.Println(string(res1B))

	//可以在结构体字段声明标签来自定义编码的JSON数据键名称
	res2D := &Response2{
		Page:1,
		Fruits:[]string{"apple", "peach", "pear"},
	}
	res2B, _ := json.Marshal(res2D)
	fmt.Println(string(res2B))

	//JSON数据解码
	byt := []byte(`{"num":6.13, "strs":["a", "b"]}`)

	//提供一个JSON包可以存放解码数据的变量
	var dat map[string]interface{}

	//实际的解码与错误检查
	if err := json.Unmarshal(byt, &dat); err != nil {
		panic(err)
	}
	fmt.Println(dat)

	//为了使用解码map中的值，可以将他们进行适当的类型转化
	//将num的值转化为float64
	num := dat["num"].(float64)
	fmt.Println(num)

	//访问嵌套的值需要一系列的转化
	strs := dat["strs"].([]interface{})
	str := strs[0].(string)
	fmt.Println(str)

	//解码JSON值到自定义类型
	//这个功能的好处：可以为程序带来额外的类型安全加强，并且消除访问数据时的类型断言
	str1 := `{"page": 1, "fruits": ["apple", "pear"]}`
	res := Response2{}
	json.Unmarshal([]byte(str1), &res)
	fmt.Println(res)
	fmt.Println(res.Fruits[0])

	//可以和os.Stdout一样，直接将JSON编码输出至os.Writer流中，或者作为HTTP响应体
	enc := json.NewEncoder(os.Stdout)
	d := map[string]int{"apple": 5, "lettuce": 7}
	enc.Encode(d)
}
```
