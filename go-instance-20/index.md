# Go Instance 20

<!--more-->
## SHA1散列
SHA1散列经常用生成二进制文件或者文本块的短标识。例如：Git版本控制系统大量使用SHA1来标识受版本控制的文件和目录。
```go
package main

import (
	"crypto/sha1"
	"fmt"
)

func main() {
	s := "sha1 this string"
	//产生一个散列值的方式是sha1.New(),sha1.Write(bytes),sha1.sum(bytes)
	h := sha1.New()

	//写入要处理的字节。如果是字符串，需使用[]byte()强制转换成字节数组
	h.Write([]byte(s))

	//这个用来得到最终的散列值的字符切片
	//Sum的参数为现有的字符切片追加额外的字节切片：一般不需要
	sum := h.Sum(nil)

	//sha1值经常以16进制输出
	fmt.Println(h)
	fmt.Println(sum)
	fmt.Printf("%x\n", sum)
}
```
## Base64编码
Go提供内建的base64编码
```go
package main

import (
	"encoding/base64"
	"fmt"
)

func main() {
	//将要编解码的字符串
	data := "abc123!?$*&()'-=@~"

	//Go同时支持标准的和URL兼容的base64格式
	//编码需要使用[]byte类型参数，故要将字符串转成字节型
	s := base64.StdEncoding.EncodeToString([]byte(data))
	fmt.Println(s)

	//解码可能会发生错误，如果不确定，需要进行错误检查
	bys, _ := base64.StdEncoding.DecodeString(s)
	fmt.Println(string(bys))
	fmt.Println()

	//使用URL兼容的base64格式进行编码解码
	s1 := base64.URLEncoding.EncodeToString([]byte(data))
	fmt.Println(s1)
	bys2, _ := base64.URLEncoding.DecodeString(s1)
	fmt.Println(string(bys2))
}
```
