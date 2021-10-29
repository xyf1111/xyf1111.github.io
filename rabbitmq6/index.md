# RabbitMQ6


## 远程过程调用(RPC)

在第二个教程中，我们学会了如何使用工作队列在多个worker之间分配耗时的任务。

但是，如果我们需要在远程计算机上运行函数并等待结果怎么办？好吧，那是一个不同的故事。这种模式通常称为"远程调用"或*RPC*。

在本教程中，我们将使用RabbitMQ构建一个RPC系统：客户端和可伸缩RPC服务器。由于我们没有值得分配的定时任务，因此我们将将构建一个虚拟的RPC服务，该服务返回斐波那契数。

```
有关RPC的说明

尽管RPC是计算中非常常见的模式，但它经常受到批评。

当程序员不知道函数调用是本地的还是缓慢的RPC时，就会出现问题。这样的混乱会导致系统变幻莫测，并给调试增加了不必要的复杂性。滥用RPC可能会导致无法维护的意大利面条式代码而不是简化软件。

牢记这一点，请考虑以下建议：
1.确定那个函数调用是本地的，那个是远程的
2.为你的系统编写文档。明确组件之间的依赖关系
3.处理错误情况。当RPC服务器长时间关闭时，客户端应如何处理
```

## 回调队列

通常，通过RabbitMQ进行RPC很容易。客户端发送请求消息，服务端发送相应消息。为了接受响应，我们需要发送带有"回调"队列地址的请求。我们可以使用默认队列。让我们尝试一下：

```go
q, err := ch.QueueDeclare(
    // 不指定队列名，默认使用随机生成的队列名
    "",
    // durable
    false,
    // delete when unused
    false,
    // exclusive
    true,
    // noWait
    false,
    // arguments
    nil,
)

err = ch.Publish(
    // exchange
    "",
    // routing key
    "rpc_queue",
    // mandatory
    false,
    // immediate
    false,
    amqp.Publishing{
        ContentType:    "text/plain",
        CorrelationId:  corrId,
        // 在这里指定callback队列名，也是在这个队列等回复
        ReplyTo:        q.Name,
        Body:           []byte(strconv.Itoa(n)),
    }
)
```

```markdown
消息属性

AMQP 0-9-1协议预定义了消息附带的14个属性集。除以下属性外，大多数属性很少用：

1. persistent:将消息标记为持久性(值为true)或瞬态(false)。你可能还记得第二个教程中的此属性。
2. content_type:用于描述编码的mime类型。例如，对于经常使用的JSON编码，将此属性设置为application/json是一个好习惯
3. reply_to:常用于命名回调队列
4. correlation_id:有助于将RPC响应与请求相关联
```

## 关联ID (Correlation Id)
在上面介绍的方法中，我们建议每个RPC请求创建一个回调队列。这是相当低效的，但是幸运的是，有一个更好的方法--让我们为每个客户端创建一个回调队列。

这就引发了一个新问题，在该队列中收到响应后，尚不清楚属于哪个请求。这个时候就应该使用correlation_id这个属性了。针对每个请求我们将其设置一个唯一值。随后，当我们在回调队列中收到消息时，我们将查看该属性，并基于这个属性将响应与请求进行匹配。如果我们看到未知的correlation_id值，则可以放心地丢弃该消息--它不属于我们的请求。

你可能会问，为什么我们应该忽略回调队列中的未知消息，而不是报错而失败？这是由于服务器端可能出现竞争状况。尽管可能性不大，但RPC服务器可能会在向我们发送答案之后，在发送请求的确认消息之前死亡。如果发生这种情况，重新启动RPC服务器将再次处理该请求。这就是为什么在客户端上我们必须妥善处理重复的响应，并且理想情况下RPC应该是幂等的。

## 总结

我们的RPC工作流程如下：
- 客户端启动时，它将创建一个匿名排他回调队列
- 对于RPC请求，客户端发送一条消息，该消息具有两个属性：reply_to(设置为回调队列)和correlation_id(设置为每个请求的唯一值)
- 该请求被发送到rpc_queue队列
- RPC工作程序(又名：服务器)正在等待该队列上的请求。当出现请求时，它会完成计算工作并把结果作为消息使用replay_to字段中的队列发回给客户端。
- 客户端等待回调队列上的数据。出现消息时，它将检查correlation_id属性。如果它与请求中的值匹配，则将响应返回给应用程序。

## 完整示例
斐波那契函数：
```go
func fib(n int) int {
    if n == 0 {
        return 0
    } else if n == 1 {
        return 1
    } else {
        return fib(n-1) + fib(n-2)
    }
}
```
声明我们的斐波那契函数。它仅假设有效的正整数输入。(不要指望这种方法适用于大量用户，它可能是最慢的递归实现)

我们的RPC服务器rpc_server.go的代码如下所示：

```go
package main

import (
    "log"
    "strconv"

    "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
    if err != nil {
        log.Fatalf("%s: %s", msg, err)
    }
}

func fib(n int) int {
    if n == 0 {
        return 0
    } else if n == 1 {
        return 1
    } else {
        return fib(n-1) + fib(n-2)
    }
}

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    failOnError(err, "Failed to connect to RabbitMQ")
    defer conn.Close()

    ch, err := conn.Channel()
    failOnError(err, "Failed to open a channel")
    defer ch.Close()

    q, err := ch.QueueDeclare(
        // name
        "rpc_queue",
        // durable
        false,
        // delete when unused
        false,
        // exclusive
        false,
        // no-wait
        false,
        // arguments
        nil,
    )
    failOnError(err, "Failed to declare a queue")

    q, err := ch.Qos(
        // prefetch count
        1,
        // prefetch size
        0,
        // global
        false,
    )
    failOnError(err, "Failed to set Qos")

    msgs, err := ch.Consume(
        // queue
        q.Name,
        // consumer
        "",
        // auto-ack
        false,
        // exclusive
        false,
        // no-local
        false,
        //  no-wait
        false,
        // args
        nil,
    )
    failOnError(err, "Failed to register a consumer")

    forever := make(chan bool)

    go func() {
        for d := range msgs {
            n, err := strconv.Atoi(string(d.body))
            failOnError(err, "Failed to convert body to integer")

            log.Printf(" [.] fib(%d)", n)
            response := fib(n)
            err = ch.Publish(
                "",
                d.ReplyTo,
                false,
                false,
                amqp.Publishing{
                    ContentType:    "text/plain",
                    CorrelationId:  d.CorrelationId,
                    Body:           []byte(strconv.Itoa(response)),
                }
            )
            failOnError(err, "failed to publish a message")

            d.Ask(false)
        }
    }()
    log.Printf(" [*] Awaiting RPC requests")
    <-forever
}
```

服务器代码非常简单：

- 和往常一样，我们首先建立连接，通道并声明队列
- 我们可能要运行多个服务器进程。为了将负载平均分配给多个服务器，我们需要在通道上设置prefetsh设置
- 我们使用Channel.Consume获取去队列，我们从队列中接收消息。然后我们进入goroutine进行工作，并将响应发送回去。
  
我们的RPC客户端rpc_client的代码

```go
package main

import (
    "log"
    "math/rand"
    "os"
    "strconv"
    "strings"
    "time"

    "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
    if err != nil {
        log.Fatalf("%s: %s", msg, err)
    }
}

func randomString(l int) string {
    bytes := make([]byte, l)
    for int i := 0; i < l; i++ {
        bytes[i] := byte(randInt(65, 90))
    }
    return string(bytes)
}

func randInt(min int, max int) int {
    return min + rand.Intn(max-min)
}

func fibonacciRPC(n int) (res int, err error) {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    failOnError(err, "Failed to connect to RabbitMQ")
    defer conn.Close()

    ch, err := conn.Channel()
    failOnError(err, "Failed to open a channel")
    defer ch.Close()

    q, err := ch.QueueDeclare(
        // name
        "",
        // durable
        false,
        // delete when unused
        false,
        // exclusive
        true,
        // noWait
        false,
        // arguments
        nil,
    )
    failOnError(err, "Failed to declare a queue")

    msg, err := ch.Consume(
        // queue
        q.Name,
        // consumer
        "",
        // auto-ack
        true,
        // exclusive
        false,
        // no-local
        false,
        // no-wait
        false,
        // args
        nil,
    )
    failOnError(err, "Failed to register a consumer")
    
    corrId := randomString(32)

    err = ch.Publish(
        "",
        "rpc_queue",
        false,
        false,
        amqp.Publishing{
            ContentType: "text/plain",
            CorrelationId:  corrId,
            ReplyTo:    q.Name,
            Body:   []byte(strconv.Itoa(n)),
        }
    )
    failOnError(err, "Failed to publish a message")

    for d := range msgs {
        if corrId == d.CorrelationId {
            res, err := strconv.Atoi(string(d.Body))
            failOnError(err, "Failed to convert body to integer")
            break
        }
    }
    return
}

func main() {
    rand.Seed(time.Now().UTC().UnixNano())
    n := bodyFrom(os.Args)
    log.Printf(" [X] Requesting fib (%d)", n)
    res, err := fibonacciRPC(n)
    failOnError(err, "Failed to handle RPC request")

    log.Printf(" [.] Got %d", res)
}

func bodyFrom(args []string) int {
    var s string
    if (len(args) < 2) || args[1] == "" {
        s = "30"
    } else {
        s = strings.Join(args[1:], " ")
    }
    n, err := strconv.Atoi(s)
    failOnError(err, "Failed to convert arg to integer")
    return n
}
```

这里介绍的设计不是RPC服务唯一可能的实现，但是它具有一些重要的特点：

- 如果RPC服务器太慢，则可以通过运行另一台RPC服务器来进行扩展。尝试在新控制台中运行另一个rpc_server.go
- 在客户端，RPC只需要发送和接收一条消息。结果，RPC客户端只需要一个网络往返就可以处理单个RPC请求

我们的代码仍然非常简单，并且不会尝试解决更复杂（但很重要）的问题，例如：

- 如果没有服务器在运行，客户端如何反应？
- 客户端是否应该为RPC设置某种超时时间？
- 如果服务器发生异常并引发故障，是否应该将其转发给客户端？
- 在处理之前防止无效的传入消息（例如检查边界，类型）。
