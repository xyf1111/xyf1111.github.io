# Go测试从零到溜1-单元测试


## Go语言测试

### go test工具

Go语言中的测试依赖`go test`命令。编写测试代码和编写普通的Go代码过程是类似的，并不需要学习新的语法、规则和工具。

`go test`命令是一个按照一定约定和组织的测试代码的驱动程序。在包目录内，所有以`_test.go`为后缀名的源代码文件都是`go test`测试的一部分，不会被`go build`编译到最终可执行文件中。

在`*_test.go`文件中有三种类型的函数，单元测试函数、基准测试函数和示例函数。

|类型|格式|作用|
|---|---|---|
|测试函数|函数名前缀为Test|测试程序的一些逻辑行为是否正确|
|基准函数|函数名前缀为Benchmark|测试函数的性能|
|示例函数|函数名前缀为Example|为文档提供示例文档|

`go test`命令会遍历所有的`*_test.go`文件中符合上述命名规则的函数，然后生成一个临时的main包用于调用相应的测试函数，然后构建并运行、报告测试结果，最后清理测试中产生的临时文件。

### 单元测试函数

**格式**

每个函数必须导入`testing`包，测试函数的基本格式(签名)如下：

```go
func TestName(t *testing.T) {
    // ...
}
```

测试函数的名字必须以`Test`开头，可选的后缀名必须以大写字母开头，举几个例子：

```go
func TestAdd(t *testing.T) {}
func TestSum(t *testing.T) {}
func TestLog(t *testing.T) {}
```

其中参数`t`用于报告测试失败和附加的日志信息。`testing.T`拥有的方法如下：

```go
func (c *T) Cleanup(func())
func (c *T) Error(args ...interface{})
func (c *T) Errorf(format string, args ...interface{})
func (c *T) Fail()
func (c *T) FailNow()
...
```
### 单元测试示例

就像细胞是构成我们身体的基本单位，一个软件程序也是由很多单元组件构成的。单元组件可以是函数、结构体、方法和最终用户可能依赖的任意东西。总之我们需要确保这些 组件是能够正常运行的。单元测试是一些利用各种方法测试单元组件的程序，它会将结果与预期输出进行比较。

接下来，我们在`base_demo`包中定义了一个`Split`函数，具体实现如下：

```go
package base_demo

import "strings"

// Split 把字符串s按照指定的分隔符sep进行分割返回字符串切片
func Split(s, sep string) (result []string) {
    i := strings.Index(s, sep)

    for i > -1 {
        result = append(result, s[:i])
        s = s[i+1:]
        i = strings.Index(s, sep)
    }

    result = append(result, s)
    return
}
```

在当前目录下我们创建一个`split_test.go`的测试文件，并定义一个测试函数如下：

```go
package split

import(
    "reflect"
    "testing"
)

// 函数名必须以Test开头，必须接收一个*testing.T类型参数
func TestSplit(t *testing.T) {
    // 程序输出结果
    got := Split("a:b:c", ":")
    // 期望的结果
    want := []string{"a", "b", "c"}
    // 因为slice不能比较直接，借助反射包中的方法比较
    if !reflect.DeepEqual(want, got) {
        // 测试失败输出错误提示
        t.Errorf("expected: %v, got: %v", want, got)
    }
}
```

在当前路径下执行`go test`命令，可以看见输出结果

### go test -v

一个测试用例有点单薄，我们再编写一个测试使用多个字符切割字符串的例子，在`split_test.go`中添加如下测试函数：

```go
func TestSplitWithComplexSep(t *testing.T) {
    got := Split("abcd", "bc")
    want := []string{"a", "d"}
    if !reflect.DeepEqual(want, got) {
        t.Errorf("expected: %v, got: %v", want, got)
    }
}
```

现在我们有多个测试用例了，为了能更好的在输出结果中看见每一个测试用例的执行情况，我们可以为`go test`命令添加`-v`参数，让它输出完整的测试结果

从输出结果我们能清楚的看到`TestSplitWithComplexSep`这个测试用例没有测试通过。

### go test -run

单元测试的结果表明`split`这个函数的实现并不可靠，没有考虑到传入的sep参数是多个字符的情况，下面修复这个bug：

```go
package base_demo

import "strings"

// Split 把字符串s按照指定的分隔符sep进行分割返回字符串切片
func Split(s, sep string) (result []string) {
    i := strings.Index(s, sep)

    for i > -1 {
        result = append(result, s[:i])
        // 这里使用len(sep)获取sep的长度
        s = s[i+len(sep):]
        i = strings.Index(s, sep)
    }

    result = append(result, s)
    return
}
```

在执行`go test`命令的时候可以添加`-run`参数，它对应的是一个正则表达式，只有函数名匹配上的测试函数才会被`go test`命令执行

例如通过给`go test`添加`-run=Sep`参数来告诉它本次测试只运行TestSplitWithComplexSep这个测试用例

### 回归测试

修改代码后仅仅执行那些失败的测试用例或新引入的测试用例是错误且危险的，正确的做法应该是完整运行所有的测试用例，保证不会因为修改代码而引入新的问题

通过这个示例我们可以看到，有了单元测试就能够在代码改动后快速进行回归测试，极大地提高开发效率和保证代码的质量。

### 跳过某些测试用例

为了节省时间支持在单元测试时跳过某些耗时的测试用例

```go
func TestTimeConsuming(t *testing.T) {
    if testing.Short() {
        t.Skip("Short 模式下会跳过该测试用例")
    }
}
```

当执行`go test short`时就不会执行上面的TestTimeConsuming测试用例

### 子测试

在上面的示例中我们为每一个测试数据编写一个测试函数，而通常单元测试中需要多组测试数据保证测试的效果。Go1.7+中新增了子测试，支持在测试函数中使用t.Run执行一组测试用例，这样就不需要为不用的测试数据定义多个测试函数了

```go
func TestXXX(t *testing.T) {
    t.Run("case1", func(t *testing.T){})
    t.Run("case2", func(t *testing.T){})
    t.Run("case3", func(t *testing.T){})
}
```

### 表格驱动测试

**介绍**

表格驱动测试不是工具、包或其他任何东西，它只是编写更清晰测试的一种方式和视角

编写好的测试并非易事，但在许多情况下，表格驱动测试可以涵盖很多方面：表格里的每一个条目都是一个完整的测试用例，包含输入和预期结果，有时还包含测试名称等附加信息，以使测试输出易于阅读。

使用表格驱动测试能够很方便的维护多个测试用例，避免在编写单元测试时频繁的复制粘贴。

表格驱动测试的步骤通常是定义一个测试用例表格，然后遍历表格，并使用`t.Run`对每一个条目执行必要的测试。

**示例**

官方标准库中有很多表格驱动测试的示例，例如fmt包中就有如下测试代码：

```go
var flagtests = []struct {
	in  string
	out string
}{
	{"%a", "[%a]"},
	{"%-a", "[%-a]"},
	{"%+a", "[%+a]"},
	{"%#a", "[%#a]"},
	{"% a", "[% a]"},
	{"%0a", "[%0a]"},
	{"%1.2a", "[%1.2a]"},
	{"%-1.2a", "[%-1.2a]"},
	{"%+1.2a", "[%+1.2a]"},
	{"%-+1.2a", "[%+-1.2a]"},
	{"%-+1.2abc", "[%+-1.2a]bc"},
	{"%-1.2abc", "[%-1.2a]bc"},
}
func TestFlagParser(t *testing.T) {
	var flagprinter flagPrinter
	for _, tt := range flagtests {
		t.Run(tt.in, func(t *testing.T) {
			s := Sprintf(tt.in, &flagprinter)
			if s != tt.out {
				t.Errorf("got %q, want %q", s, tt.out)
			}
		})
	}
}
```

通常表格是匿名结构体的切片，可以定义结构体或使用已经存在的结构进行结构体数组声明。name属性用来描述特定的测试用例。

接下来让我们试着自己编写表格驱动测试：

```go
func TestSplitAll(t *testing.T) {
	// 定义测试表格
	// 这里使用匿名结构体定义了若干个测试用例
	// 并且为每个测试用例设置了一个名称
	tests := []struct {
		name  string
		input string
		sep   string
		want  []string
	}{
		{"base case", "a:b:c", ":", []string{"a", "b", "c"}},
		{"wrong sep", "a:b:c", ",", []string{"a:b:c"}},
		{"more sep", "abcd", "bc", []string{"a", "d"}},
		{"leading sep", "沙河有沙又有河", "沙", []string{"", "河有", "又有河"}},
	}
	// 遍历测试用例
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) { // 使用t.Run()执行子测试
			got := Split(tt.input, tt.sep)
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("expected:%#v, got:%#v", tt.want, got)
			}
		})
	}
}
```

在终端执行`go test -v`， 会得到测试输出结果

**并行测试**

表格驱动测试中通常会定义比较多的测试用例，而Go语言又天生支持并发，所以很容易就发挥自身并发优势将表格驱动测试并行化。想要在单元测试过程中使用并行测试，可以像下面的代码示例中那样通过添加`t.Parallel()`来实现

```go
func TestSplitAll(t *testing.T) {
	t.Parallel()  // 将 TLog 标记为能够与其他测试并行运行
	// 定义测试表格
	// 这里使用匿名结构体定义了若干个测试用例
	// 并且为每个测试用例设置了一个名称
	tests := []struct {
		name  string
		input string
		sep   string
		want  []string
	}{
		{"base case", "a:b:c", ":", []string{"a", "b", "c"}},
		{"wrong sep", "a:b:c", ",", []string{"a:b:c"}},
		{"more sep", "abcd", "bc", []string{"a", "d"}},
		{"leading sep", "沙河有沙又有河", "沙", []string{"", "河有", "又有河"}},
	}
	// 遍历测试用例
	for _, tt := range tests {
		tt := tt  // 注意这里重新声明tt变量（避免多个goroutine中使用了相同的变量）
		t.Run(tt.name, func(t *testing.T) { // 使用t.Run()执行子测试
			t.Parallel()  // 将每个测试用例标记为能够彼此并行运行
			got := Split(tt.input, tt.sep)
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("expected:%#v, got:%#v", tt.want, got)
			}
		})
	}
}
```

这样我们执行`go test -v`的时候就会看到每个测试用例并不是按照我们定义的顺序执行的，而是互相并行的。

**使用工具生成测试代码**

社区里有很多自动生成表格驱动测试函数的工具，比如`gotests`等，很多编辑器如Goland也支持快速生成测试文件。这里简单演示一下`gotests`的使用。

安装

```bash
go get -u github.com/cweill/gotests/...
```

执行

```bash
gotests -all -w split.go
```

上面的命令表示，为`split.go`文件的所有函数生成测试代码至`split_test.go`文件(目录下如果事先存在这个文件就不再生成了)

代码格式与我们上面的类似，只需要在TODO位置添加我们的测试逻辑就可以了

### 测试覆盖率

测试覆盖率是指代码被测试套件覆盖的百分比。通常我们使用的语句都是有覆盖率，也就是在测试中至少被运行一次的代码占总代码的比例。在公司内部一般会要求测试覆盖率达到80%左右。

Go提供内置功能来检查你的代码覆盖率，即使用`go test -cover`来查看测试覆盖率

Go还提供了一个额外的`-coverprofile`参数，用来将覆盖率相关的记录信息输出到一个文件。

```bash
go test -cover -coverprofil=c.out
```

上面的命令会将覆盖率相关的信息输出到当前文件夹下面的`c.out`文件中

然后我们执行`go tool cover -html=c.out`，使用`cover`工具来处理生成的记录信息，该命令会打开本地的浏览器窗口生成一个html报告

## testify/assert

testify是一个社区非常流行的Go单元测试工具包，其中使用最多的就是它提供的断言工具--`testify/assert`或`testify/require`

### 安装

```bash
go get github.com/stretchr/testify
```

### 使用示例

我们在写单元测试的时候，通常需要使用断言来校验测试结果，但是由于Go语言官方没有提供断言，所以我们会写出很多的`if...else...`语句。而`testify/assert`为我们提供了很多常用的断言函数，并且能够输出友好，易于阅读的错误描述信息。

比如之前在`TestSplit`测试函数中就使用了`reflect.DeepEqual`来判断期待结果与实际结果是否一致。

```go
t.Run(tt.name, func(t *testing.T) { // 使用t.Run()执行子测试
	got := Split(tt.input, tt.sep)
	if !reflect.DeepEqual(got, tt.want) {
		t.Errorf("expected:%#v, got:%#v", tt.want, got)
	}
})
```

使用`testify/assert`之后就能将上述判断过程简化如下：

```go
t.Run(tt.name, func(t *testing.T) { // 使用t.Run()执行子测试
	got := Split(tt.input, tt.sep)
	assert.Equal(t, got, tt.want)  // 使用assert提供的断言函数
})
```

当我们有多个断言语句时，还可以使用`assert := assert.New(t)`创建一个assert对象，它拥有前面所有的断言方法，只是不需要再传入Testing.T参数了。

## 总结

本文介绍了Go语言单元测试的基本用法，通过为Split函数编写单元测试的真实案例，模拟了日常开发过程中的场景，一步一步详细介绍了表格驱动测试、回归测试和常用的测试工具`testify/assert`的使用。
