# Go Instance 12

<!--more-->
## sort自带排序
Go 的 sort 包实现了内置和用户自定义数据类型的排序功能
```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	//排序方法是正对内置数据类型的；这里是一个字符串的例子。
	// 注意排序是原地更新的，所以他会改变给定的序列并且不返回一个新值。
	strs := []string{"c", "b", "d", "a"}
	sort.Strings(strs)
	fmt.Println(strs)

	//一个 int 排序的例子。
	ints := []int{4,2,3,1}
	sort.Ints(ints)
	fmt.Println(ints)

	//我们也可以使用 sort 来检查一个序列是不是已经是排好序的。
	sorted := sort.IntsAreSorted(ints)
	fmt.Println(sorted)
}
```
## 使用函数自定义排序
使用和集合的自然排序不同的方法对集合进行排序。例如按字母的长度而不是首字母顺序对字符串排序。
```go
package main

import (
	"fmt"
	"sort"
)

//为了在Go中使用自定义函数进行排序，需要定义一个对应类型
//这里创建一个为内置[]string类型的别名ByLength类型
type ByLength []string

//在类型中实现了sort.Interface 的Len，Less，Swap方法
//这样就可以使用sort包内通用的Sort方法了，Len和Swap通常在各个类型中差不多
//Less将控制实际的自定义排序逻辑
//在这个例子中按字符串的长度来进行排序，所以这里用到了len(s[i])和len(s[j])
func (s ByLength) Len() int {
	return len(s)
}

func (s ByLength) Less(i int, j int) bool {
	return len(s[i]) < len(s[j])
}

func (s ByLength) Swap(i int, j int) {
	s[i], s[j] = s[j], s[i]
}

func main() {
	//将原始的fruits转型成ByLength来实现自定义排序
	//然后对这个转型的切片使用sort.Sort方法
	fruits := []string{"peach", "banana", "kiwi"}
	sort.Sort(ByLength(fruits))
	fmt.Println(fruits)
}
```
