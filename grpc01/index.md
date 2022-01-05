# Grpc01


## RPC是什么

在分布式计算，远程过程调用(英语：Remote Procedure Call，缩写为RPC)是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一个地址空间(通常为一个开放网络的一台计算机)的子程序，而程序员就像调用本地程序一样，无需额外地为这个交互作用编程(无需关注细节)。RPC是一种服务器-客户端(Client/Server)模式，经典实现是通过一个`发送请求-接收回应`进行信息交互的系统

## gRPC是什么

`gRPC`是一种现代化开源的高性能RPC框架，能够运行于任意环境之中。最初由谷歌进行开发。它使用HTTP/2作为传输协议。

在gRPC里，客户端可以像调用本地方法一样直接调用其他机器上的服务端应用程序的方法，帮助你更容易创建分布式应用程序和服务。和许多RPC系统一样，gRPC是基于定义一个服务，指定一个可以远程调用的带有参数和返回类型的方法。在服务端程序中实现这个接口并且运行gRPC服务处理客户端调用。在客户端，有一个stub提供和服务端相同的方法。

![""](/images/grpc.svg "")

## 为什么要用gRPC

使用gRPC，我们可以一次性的在一个`.proto`文件中定义服务并使用任何支持它的语言去实现客户端和服务端，反过来，它们可以应用在各种场景中，从Google的服务器到你自己的平板电脑--gRPC帮你解决了不同语言及环境间通信的复杂性。使用`protocol buffers`还能获得其他好处，包括高效的序列号，简单的IDL以及容易进行接口更新。总之一句话，使用gRPC能让我们更容易编写跨语言的分布式代码

## 安装gRPC

### 安装gRPC

```bash
go get -u google.golang.org/grpc
```

### 安装Protocol Buffers V3

安装用于生成gRPC服务代码的协议编译器，最简单的方法是从下面的链接：`https://github.com/google/protobuf/releases`下载适合你平台的预编译好的二进制文件(`protoc-<version>-<platform>.zip`)

下载完，执行下面步骤：
1. 解压下载好的文件
2. 把`protoc`二进制文件的路径加到环境变量中

接下来执行下面的命令安装protoc的Go插件

```bash
go get -u github.com/golang/protobuf/protoc-gen-go
```

编译插件`protoc-gen-go`将会安装到`$GOBIN`，默认是`$GOPATH/bin`，它必须在你的`$PATH`中以便协议编译器`protoc`能够找到它

## gRPC开发分三步

1. 编写`.proto`文件，生成指定语言源代码
2. 编写服务端代码
3. 编写客户端代码

## gRPC入门实例

### 编写proto代码
gRPC是基于Protocol Buffers。

`Protocol Buffers`是一种与语言无关，与平台无关的可扩展机制，用于序列化结构化数据。使用`Protocol Buffers`可以一次定义结构化的数据，然后可以使用特殊生成的源代码轻松地在各种数据流中使用各种语言编写和读取结构化数据。

关于`Protocol Buffers`的教程可以自行网上搜索。

```proto
syntax = "proto3"; // 版本声明，使用Protocol Buffers v3版本

package pb; // 包名


// 定义一个打招呼服务
service Greeter {
    // SayHello 方法
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 包含人名的一个请求消息
message HelloRequest {
    string name = 1;
}

// 包含问候语的响应消息
message HelloReply {
    string message = 1;
}
```

执行下面命令，生成go语言源代码

```bash
protoc -I helloworld/ helloworld/pb/helloworld.proto --go_out=plugins=grpc:helloworld
```

在`gRPC_demo/helloworld/pb`目录下会生成`helloworld.pb.go`文件

### 编写Server端Go代码

```go
package main

import (
	"fmt"
	"net"

	pb "gRPC_demo/helloworld/pb"
	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

type server struct{}

func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func main() {
	// 监听本地的8972端口
	lis, err := net.Listen("tcp", ":8972")
	if err != nil {
		fmt.Printf("failed to listen: %v", err)
		return
	}
	s := grpc.NewServer() // 创建gRPC服务器
	pb.RegisterGreeterServer(s, &server{}) // 在gRPC服务端注册服务

	reflection.Register(s) //在给定的gRPC服务器上注册服务器反射服务
	// Serve方法在lis上接受传入连接，为每个连接创建一个ServerTransport和server的goroutine。
	// 该goroutine读取gRPC请求，然后调用已注册的处理程序来响应它们。
	err = s.Serve(lis)
	if err != nil {
		fmt.Printf("failed to serve: %v", err)
		return
	}
}
```

将上面的代码保存到`gRPC_demo/helloworld/server/server.go`文件中，编译并执行

```bash
cd helloworld/server
go build
./server
```

### 编写Client端Go代码

```go
package main

import (
	"context"
	"fmt"

	pb "gRPC_demo/helloworld/pb"
	"google.golang.org/grpc"
)

func main() {
	// 连接服务器
	conn, err := grpc.Dial(":8972", grpc.WithInsecure())
	if err != nil {
		fmt.Printf("faild to connect: %v", err)
	}
	defer conn.Close()

	c := pb.NewGreeterClient(conn)
	// 调用服务端的SayHello
	r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name: "q1mi"})
	if err != nil {
		fmt.Printf("could not greet: %v", err)
	}
	fmt.Printf("Greeting: %s !\n", r.Message)
}
```

将上述代码保存到`gRPC_demo/helloworld/client/client.go`文件中，编译并执行

```bash
cd helloworld/client/
go build
./client
```

得到输出如下(注意要县启动server端再启动client端):

```bash
$ ./client 
Greeting: Hello q1mi!
```

### gRPC跨语言调用

接下来，我们演示一下如何使用gRPC实现跨语言的RPC调用

我们使用`Python`语言编写`Client`，然后向上面使用`Go`语言编写的`server`发送RPC请求

### 生成Python代码

在`gRPC_demo`目录执行下面的命令

```bash
python -m grpc_tools.protoc -I helloworld/pb/ --python_out=helloworld/client/ --grpc_python_out=helloworld/client/ helloworld/pb/helloworld.proto
```

上面的命令会在`gRPC_demo/helloworld/client/`目录生成如下两个Python文件：

```bash
helloworld_pb2.py
helloworld_pb2_grpc.py
```

### 编写Python版Client

在`gRPC_demo/helloworld/client/`目录创建`client.py`文件，内容如下

```python
# coding=utf-8

import logging

import grpc

import helloworld_pb2
import helloworld_pb2_grpc


def run():
    # 注意(gRPC Python Team): .close()方法在channel上是可用的。
    # 并且应该在with语句不符合代码需求的情况下使用。
    with grpc.insecure_channel('localhost:8972') as channel:
        stub = helloworld_pb2_grpc.GreeterStub(channel)
        response = stub.SayHello(helloworld_pb2.HelloRequest(name='q1mi'))
    print("Greeter client received: {}!".format(response.message))


if __name__ == '__main__':
    logging.basicConfig()
    run()
```

将上面的代码保存执行，得到输出结果如下

```bash
gRPC_demo $ python helloworld/client/client.py 
Greeter client received: Hello q1mi!
```

这里我们就实现了，使用python代码编写的client去调用Go语言版本的server。


