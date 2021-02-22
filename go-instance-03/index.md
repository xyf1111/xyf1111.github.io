# Go Instance 03

<!--more-->
## 字符操作
### 案例1
统计一个文本文件的字符个数
```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
)
//定义一个结构体保存统计结果
type charCount struct {
	//英文个数
	chCount		int
	//空格个数
	spaceCount	int
	//数字个数
	numCount	int
	//其他个数
	otherCount	int
}

func main() {
	/**
	思路：打开一个文件，创建一个Reader
	每读取一行，统计字符保存到结构体中
	 */
	file, err := os.Open("fileOperations/abc.txt")
	if err != nil {
		fmt.Println("file open err:", err)
	}
	reader := bufio.NewReader(file)
	fileC := charCount{}
	//循环读取file内容
	for {
		s, err := reader.ReadString('\n')
		//为了兼容中文转化为rune
		for _, r := range []rune(s) {
			switch {
			case r >= 'a' && r <= 'z':
				fallthrough
			case r >= 'A' && r <= 'Z':
				fileC.chCount++
			case r > '0' && r < '9':
				fileC.numCount++
			case r == ' ' || r == '\t':
				fileC.spaceCount++
			default:
				fileC.otherCount++
			}
		}
		if err == io.EOF {
			break
		}
	}

	fmt.Printf("file has chcount: %d, numcount: %d, spaceCount: %d, otherCount: %d", fileC.chCount, fileC.numCount, fileC.spaceCount, fileC.otherCount)
}
```
## 将数据序列化成json字符串
### 将数据(结构体，Map，Slice，基础数据类型)序列化成json字符串
json序列化是指将有key-value结构的数据类型(比如：结构体，Map，Slice)序列化成json字符串的操作
```go
package main

import (
	"encoding/json"
	"fmt"
)

type Hero struct {
	Name	string	`json:"hero_name"`
	Age		int		`json:"hero_age"`
	Sex		string
}
//结构体序列化
func changeStruct() {
	hero := Hero{
		Name:	"小陈",
		Age:	18,
		Sex:	"男",
	}
	//将结构体序列化
	bytes, err := json.Marshal(hero)
	if err != nil{
		fmt.Println("json marshal fail:", err)
	}
	fmt.Printf("结构体序列化后的字段：%v\n", string(bytes))
}
//Map序列化
func changeMap() {
	map1 := make(map[string]interface{})
	map1["name"] = "张无忌"
	map1["age"] = 22
	map1["address"] = "冰火岛"
	//将map序列化
	bytes, err := json.Marshal(map1)
	if err != nil {
		fmt.Println("json marshal fail:", err)
	}
	fmt.Printf("Map序列化后的字段：%v\n", string(bytes))
}
//Slice序列化
func changeSlice() {
	var slice []map[string]interface{}
	m1 := make(map[string]interface{})
	m1["name"] = "张无忌"
	m1["age"] = 25
	m1["address"] = "冰火岛"
	slice = append(slice, m1)

	m2 := make(map[string]interface{})
	m2["name"] = "张三丰"
	m2["age"] = "88"
	m2["address"] = []string{"武当山", "夏威夷"}
	slice = append(slice, m2)
	//slice序列化
	bytes, err := json.Marshal(slice)
	if err != nil {
		fmt.Println("json marshal fail:", err)
	}
	fmt.Printf("Slice序列化后的字段：%v\n", string(bytes))
}
//基本类型序列化(对基础类型序列化意义不大)
func changeFloat() {
	var f float64 = 3.14
	//对float序列化
	bytes, err := json.Marshal(f)
	if err != nil {
		fmt.Println("json marshal fail:", err)
	}

	fmt.Println("Float序列化后的字段：", string(bytes))
}

func main() {
	changeStruct()
	changeMap()
	changeSlice()
	changeFloat()
}

```
## 将json字符串反序列化成对应数据
### 将json字符串 反序列化成对应数据（比如：结构体，map，切片）
```go
package main

import (
	"encoding/json"
	"fmt"
)

type Hero struct {
	Name	string	`json:"hero_name"`
	Age		int		`json:"hero_age"`
	Sex		string
}
//将json字符串反序列化为结构体
func unmarshalStruct() {
	str := "{\"hero_name\":\"小陈\",\"hero_age\":18,\"Sex\":\"男\"}"

	var hero Hero

	err := json.Unmarshal([]byte(str), &hero)
	if err != nil {
		fmt.Println("json unmarshal fail:", err)
	}
	fmt.Printf("json反序列化为结构体：%v\n", hero)
}
//将json字符串反序列化为Map
func unmarshalMap() {
	str := "{\"name\": \"张无忌\",\"age\": 22,\"address\": \"冰火岛\"}"
	//定义一个map，反序列化的时候不需要make，因为unmarshal封装了make
	var map1 map[string]interface{}

	err := json.Unmarshal([]byte(str), &map1)

	if err != nil {
		fmt.Println("json unmarshal fail:", err)
	}
	fmt.Printf("json反序列化为Map：%v\n",map1)
}
//将json字符串反序列化为Slice
func unmarshalSlice() {
	str := "[{\"address\":\"冰火岛\",\"age\":25,\"name\":\"张无忌\"},{\"address\":[\"武当山\",\"夏威夷\"],\"age\":\"88\",\"name\":\"张三丰\"}]"
	//定义一个slice，反序列化的时候不需要make，因为unmarshal封装了make
	var slice []map[string]interface{}

	err := json.Unmarshal([]byte(str), &slice)
	if err != nil {
		fmt.Println("json unmarshal fail:", err)
	}
	fmt.Printf("json反序列化为Slice：%v\n", slice)
}

func main() {
	unmarshalStruct()
	unmarshalMap()
	unmarshalSlice()
}
```
