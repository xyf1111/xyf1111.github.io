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
	// 逐行读取file的内容
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
## 案例
```go
func sample()  {
	//1.最常用的方式，带单个循环条件
	i := 1
	for i <= 3 {
		fmt.Println(i)
		i += 1
	}
	//2.经典的初始化/条件/后续形式for循环
	for j := 7; j <= 9; j++ {
		fmt.Println(j)
	}
	//3.不带条件的for循环将一直执行，直到循环体内使用了break或者return来跳出循环
	for {
		fmt.Println("loop")
		break
	}
}
```
### Range遍历
range迭代各种各样的数据结构
```go
func loopSample2()  {
	//这里使用range来统计一个slice的元素个数，数组也采用这种方式
	nums := []int{2,4,5,3}
	sum := 0
	for _, num := range nums {
		sum += num
	}
	fmt.Println("sum: ", sum)
	//range在数组和slice中同样提供了每项的索引和值
	//不需要索引时可以使用_来忽略她
	for i, num := range nums {
		if num == 3 {
			fmt.Println("index: ", i)
		}
	}
	//range在map中迭代键值对
	kvs := map[string]string{
		"name": "cc",
		"age":	"18",
	}
	for k, v := range kvs {
		fmt.Printf("%v -> %v\n", k, v)
	}
	//range在字符串中迭代Unicode编码
	for i, v := range "go" {
		fmt.Println(i, v)
	}
}
```
