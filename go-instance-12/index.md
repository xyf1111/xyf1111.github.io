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
