# Go Study 03

<!--more-->
## for
- for的条件里不需要括号
- for的条件里可以省略初始条件，结束条件，递增表达式
```go
sum := 0
for i := 1; i <= 100; i++ {
    sum += i
}
```
- 省略初始条件，相当于while
```go
 func convertToBin(n int) string {
	result := ""
	for ; n > 0; n /= 2 {
		lsp := n % 2
		result = strconv.Itoa(lsp) + result
	}
	return result
}
```
- 省略初始条件和递增条件，也相当于while
```go
func printFile(filename string)  {
	file, err := os.Open(filename)
	if err != nil {
		panic(err)
	}
	scanner := bufio.NewScanner(file)

	for scanner.Scan() {
		fmt.Println(scanner.Text())
	}
}
```
- 无限循环
```go
func forever()  {
	for  {
		fmt.Println("abc")
	}
}
```
## 基本语法要点回顾
- for, if后面的条件没有括号
- if条件里也可以定义变量
- 没有while
- switch不需要break，也可以直接switch多个条件
