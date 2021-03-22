# Go Instance 14

<!--more-->
## 组合函数
我们经常需要程序在数据集上执行操作，比如选择满足条件的所有想，或者将所有项通过一个自定义函数映射到一个新的集合上

在某些语言中，会习惯使用泛型。Go不支持泛型，在Go中当你的程序或数据类型有需要时，通常是以组合的方式来提供操作函数

这是一些strings切片的组合函数实例。可以使用这些例子来构建自己的函数。注意有些时候直接使用内联组合操作代码会更加清晰，而不是创建并调用一个函数
```go
package main

import (
	"fmt"
	"strings"
)

//返回目标字段s出现的第一个位置，没有返回-1
func Index(s string, arr []string) int {
	for i, v := range arr {
		if s == v {
			return i
		}
	}
	return -1
}

//如果目标s在切片中存在返回true
func Include(s string, arr []string) bool {
	return Index(s, arr) >= 0
}

//如果切片中的字符串有一个满足条件f返回true
func Any(arr []string, f func(string) bool) bool {
	for _, v := range arr {
		if f(v) {
			return true
		}
	}
	return false
}

//如果切片中的字符串所有都满足条件f返回true
func All(arr []string, f func(string) bool)	bool {
	for _, v := range arr {
		if !f(v) {
			return false
		}
	}
	return true
}

//返回一个包含所有切面中满足条件f的新切片
func Filer(arr []string, f func(string) bool) []string {
	result := make([]string, 0)
	for _, v := range arr{
		if f(v) {
			result = append(result, v)
		}
	}
	return result
}

//返回一个对原始切片中所有字符串执行函数f的切片
func Map(arr []string, f func(string) string) []string {
	result := make([]string, 0)
	for _, v := range arr {
		result = append(result, f(v))
	}
	return result
}

func main() {
	fruits := []string{"peach", "apple", "pear", "plum"}
	fmt.Println(Index("pear", fruits))
	fmt.Println(Include("grape", fruits))
	fmt.Println(Any(fruits, func(v string) bool {
		return strings.HasPrefix(v, "p")
	}))
	fmt.Println(All(fruits, func(v string) bool {
		return strings.HasPrefix(v, "p")
	}))
	fmt.Println(Filer(fruits, func(v string) bool {
		return strings.Contains(v, "e")
	}))
	//上面都是使用匿名函数，也可以使用类型正确的命名函数
	fmt.Println(Map(fruits, strings.ToUpper))
}
```
## 字符串函数
标准库的strings包提供了很多有用的字符相关的函数。这里是一些用来让你对这个包有个初步理解的例子
```go
package main

import (
	"fmt"
	s "strings"
	)

//给fmt.Println起别名，之后会经常用到
var p = fmt.Println

func main() {
	//这是一些strings中的函数例子。注意他们都是函数中的例子，不是字符串对自身的方法
	//这意味着我们需要考虑在调用时传递字符作为第一个参数进行传递
	p("Contains:	", s.Contains("test", "es"))
	p("Count:		", s.Count("test", "t"))
	p("HasPrefix:	", s.HasPrefix("test", "te"))
	p("HasSuffix:	", s.HasSuffix("test", "st"))
	p("Index:		", s.Index("test", "e"))
	p("Join:		", s.Join([]string{"a", "b"}, "-"))
	p("Repeat:		", s.Repeat("a", 5))
	p("Replace:	", s.Replace("foo", "o", "O", -1))
	p("Replace:	", s.Replace("foo", "o", "O", 1))
	p("Split:		", s.Split("a-b-c-d-e", "-"))
	p("ToLower:	", s.ToLower("TEST"))
	p("ToUpper:	", s.ToUpper("test"))
}
```
