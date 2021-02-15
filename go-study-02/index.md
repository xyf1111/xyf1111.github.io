# Go Study 02

<!--more-->
## if
- if的条件里不需要括号
```go
    func bounded(v int) int {
        if v > 100 {
            return 100
        } else if v < 0 {
            return 0
        } else {
            return v
        }
    }
```
- if的条件里可以赋值
```go
    if contents, err := ioutil.ReadFile(filename); err != nil {
		fmt.Println(err)
	} else {
		fmt.Printf("%s\n", contents)
	}
```
- if条件里赋值的变量作用域就在这个if语句里
## switch
- switch会自动break，除非使用fallthrough
```go
func eval(a, b int, op string) int {
	var result int
	switch op {
	case "+":
		result = a + b
	case "-":
		result = a - b
	case "*":
		result = a * b
	case "/":
		result = a / b
	default:
		panic("unsupported operator:" + op)
	}
	return result
}
```
- switch后可以没有表达式
```go
func grade(score int) string {
	g := ""
	switch  {
	case score < 60:
		g = "D"
	case score < 80:
		g = "C"
	case score < 90:
		g = "B"
	case score <= 100:
		g = "A"
	default:
		panic(fmt.Sprintf("Wrong score: %d", score))
	}
	return g
}
```
