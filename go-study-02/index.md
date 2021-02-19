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
## 案例
### if/else分支
if和else分支结构在Go中当然是直接了当的了
```go
func branchSample1()  {
	if 7 % 2 == 0 {
		//7是偶数
		fmt.Println("7 is even")
	} else {
		//7是奇数
		fmt.Println("7 is odd")
	}
	//也可以不要else，只用if
	if 8 % 4 == 0 {
		fmt.Println("8 is divisible by 4")
	}
	//在条件语句之前可以有一个语句，任何在这里声明的变量都可以在所有的条件分支中使用
	if num := 9; num < 0 {
		fmt.Println(num, "is negative")
	} else if num < 10 {
		fmt.Println(num, "has 1 digit")
	} else {
		fmt.Println(num, "has multiple digit")
	}
}
```
### switch分支
```go
func branchSample2()  {
	//一个基本的switch
	i := 2
	switch i {
	case 1:
		fmt.Println("one")
	case 2:
		fmt.Println("two")
	case 3:
		fmt.Println("three")
	}
	//在一个case语句中可以使用逗号分割多个表达式
	//使用了可选的default分支
	switch time.Now().Weekday() {
	case time.Saturday, time.Sunday:
		fmt.Println("It's the weekend")
	default:
		fmt.Println("It's a weekday")
	}
	//不带表达式的switch是实现if/else的另一种方式
	//这里展示了case表达式是如何使用非常量的
	t := time.Now()
	switch  {
	case t.Hour() < 12:
		fmt.Println("it's before noon")
	default:
		fmt.Println("it's after noon")
	}
}
```
