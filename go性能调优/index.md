# Go性能调优

## Go性能优化
Go语言项目中的性能优化主要有以下几个方面
- CPU profile：报告程序的CPU使用情况，按照一定频率去采集应用程序在CPU和寄存器上面的数据
- Memory Profile (Heap Profile)：报告程序的内存使用情况
- Block Profiling：报告goroutine不在运行时状态的情况，可以用来分析和查找死锁等性能瓶颈
- Goroutine Profiling：报告goroutines的使用情况，有哪些goroutine，它们的调用关系是怎样的
## 采集性能数据
Go语言内置了获取程序运行数据的工具，包括以下两个标准库：
- runtime/pprof：采集工具型应用运行数据进行分析
- net/http/pprof：采集服务型应用运行数据进行分析
pprof开启后，每个一段时间(10ms)就会收集下当前的堆栈信息，获取各个函数占用的CPU以及内存资源；最后通过对这些采样数据进行分析，形成一个性能分析报告。

注意，我们只应该在性能测试的时候才在代码中引入pprof
## 工具型应用
如果你的应用程序是运行一段时间就退出类型。那么最好的方法是在应用退出的时候把profiling的报告保存到文件中，进行分析。对于这种情况，可以使用runtime/pprof库。首先在代码中导入runtime/pprof工具：
```go
import "runtime/pprof"
```
### CPU性能分析
开启CPU性能分析
```go
pprof.StartCPUProfile(w io.Writer)
```
停止CPU性能分析
```go
pprof.StopCPUProfile()
```
应用执行结束后，就会生成一个文件，保存了我们的CPU profiling数据。得到采样数据之后，使用go tool pprof工具进行CPU性能分析
## 内存性能优化
记录程序的堆栈信息
```go
pprof.WriteHeapProfile(w io.Writer)
```
得到采样数据之后，使用go tool pprof工具进行内存性能分析。

go tool pprof默认是使用-inuse_space进行统计，还可以使用-inuse-objects查看分配对象的数量。
## 服务型应用
如果你的程序是一直运行的，比如web应用，可以使用net/http/pprof库，它能够在提供HTTP服务进行分析。如果使用了默认的http.DefaultServeMux(通常是代码直接用http.ListenAndServe("0.0.0.0:8000", nil))，只需要在你的web server端代码中按如下方式导入net/http/pprof
```go
import _ "net/http/pprof"
```
如果使用自定义的Mux，则需要手动注册一些路由规则
```go
r.HandleFunc("/debug/pprof/", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
```
如果使用的是gin框架，那么推荐使用github.com/gin-contrib/pprof，在代码中通过以下命令注册pprof相关路由
```go
pprof.Register(router)
```
不管哪种方式，你的HTTP服务都会多出/debug/pprof endpoint,可以通过(0.0.0.0:8080/debug/pprof)进行访问

这个路径下还有几个子页面：
- /debug/pprof/profile：访问这个链接会自动进行CPU profiling，持续30s，并生成一个文件供下载
- /debug/pprof/heap：Memory Profiling的路径，访问这个链接会得到一个内存Profiling结果的文件
- /debug/pprof/block：block Profiling的路径
- /debug/pprof/goroutines：运行的goroutines列表，以及调用关系
## go tool pprof命令
不管是工具型还是服务型应用，我们使用相应的pprof库获取数据之后，下一步的都要对这些数据进行分析，我们可以使用go tool pprof命令行工具。

go tool pprof最简单的使用方式为：
```go
go tool pprof [binary] [source]
```
其中：
- binary是应用的二进制文件，可以解析各种符号
- source表示profile数据的来源，可以是本地的文件，也可以是http地址
注意事项：获取的Profiling数据是动态的，要想获得有效的数据，请保证应用处于较大的负载(比如正在生成中运行的服务，或者通过其他工具模拟访问压力)。否则如果应用处于空闲状态，得到的结果可能没有任何意义。
## 具体示例
## go-torch和火焰图
火焰图(Flame Graph)是Bredan Gregg创建的一种性能分析表图，因为它的样子近似火焰而得名。上面的profiling结果也转换成火焰图，如果对火焰图比较了解可以手动来操作，不过这里我们要介绍一个工具：go-torch。这是Uber开源的一个工具，可以直接读取golang profiling数据，并生成一个火焰图的svg文件
### 安装go-torch
```go
go get -v githug.com/uber/go-torch
```
火焰图svg文件可以通过浏览器打开，它对调用图的最大优点是它是动态的：可以通过点击每个方块来zoom in分析它上面的内容。

火焰图的调用顺序从下到上，每个方块代表一个函数，它上面一层表示这个函数会调用哪些函数，方块的大小代表了占用CPU使用的长短。火焰图的配色并没有特殊的意义，默认的红、黄配色是为了更像火焰而已。

go-torch工具的使用非常简单，没有任何参数的话，它会尝试从http://localhost:8080/debug/pprof/profile获取profiling数据。它有三个常用的参数可以调整：
- -u -url：要访问的URL，这里只是主机和端口部分
- -s -suffix：pprof profile的路径，默认为/debug/pprof/profile
- -seconds：要执行profiling的时间长度，默认为30s
### 安装FlameGraph
要生成火焰图，需要事先安装FlameGraph工具，这个工具安装很简单(需要perl环境支持)，只要把对应的可执行文件加入到环境变量中即可。
1. 下载安装perl：https://www.perl.org/get.html
2. 下载FlameGraph：git clone https://github.com/brendangregg/FlameGraph.git
3. 将FrameGraph目录加入到操作系统的环境变量中
4. window只需要把go-torch/render/flamegraph.go文件中的GenerateFlameGraph按如下方式修改，然后在go-torch目录下执行go install即可
```go
// GenerateFlameGraph runs the flamegraph script to generate a flame graph SVG. func GenerateFlameGraph(graphInput []byte, args ...string) ([]byte, error) {
flameGraph := findInPath(flameGraphScripts)
if flameGraph == "" {
	return nil, errNoPerlScript
}
if runtime.GOOS == "windows" {
	return runScript("perl", append([]string{flameGraph}, args...), graphInput)
}
  return runScript(flameGraph, args, graphInput)
}
```
### 压测工具
推荐使用https://github.com/wg/wrk 或 https://github.com/adjust/go-wrk
### 使用go-torch
使用wrk进行压测
```go
go-work -n 50000 http://127.0.0.1:8080/book/list
```
在上面进行压测的同时，打开另一个终端执行
```go
go-torch -u http://127.0.0.1:8080 -t 30
```
30秒之后终端就会出现如下提示
```go
Writing svg to torch.svg
```
然后我们使用浏览器打开torch.svg就能看到火焰图了

火焰图的y轴表示cpu调用方法的先后，x轴表示在每个采样调用时间内，方法所占的时间百分比，越宽代表占据CPU时间越多。通过火焰图我们就可以更清楚的找出耗时长的函数调用，然后不断的修正代码，重新采样不断优化。

此外还可以借助火焰图分析内存性能数据：
```go
go-torch -inuse_space http://127.0.0.1:8080/debug/pprof/heap
go-torch -inuse_objects http://127.0.0.1:8080/debug/pprof/heap
go-torch -alloc_space http://127.0.0.1:8080/debug/pprof/heap
go-torch -alloc_objects http://127.0.0.1:8080/debug/pprof/heap
```
