# RabbitMQ4


## 路由
在上一教程中，我们构建了一个简单的日志记录系统。我们能够向许多接收者广播日志消息。

在本教程中，我们将它添加一个特性--我们将使它能够只订阅消息的一个子集。例如，我们将只能将错误消息定向到日志文件(以节省磁盘空间)，同时仍然能够在控制台上打印所有日志消息。

### 绑定
在前面的示例中，我们已经在创建绑定。你可能会想起以下代码：

```go
err = ch.QueueBind(
    // queue name
    q.Name,
    // routing key
    "",
    // exchange
    "logs",
    false,
    nil,
)
```

绑定是交换器和队列之间的关系。这可以简单地理解为：队列对来自此交换器的消息感兴趣。

绑定可以采用额外的routing_key参数。为了避免与Channel.Publish参数混淆，我们将其称为binding key。这是我们如何使用键创建绑定的方法：

```go
err = ch.QueueBind(
    // queue name
    q.Name,
    // routing key
    "black",
    // exchange
    "logs",
    false,
    nil,
)
```

绑定密钥的含义取决于交换器的类型。我们以前使用的fanout交换器只是忽略了这个值。

### 直连交换器
我们上一个教程中的日志系统将所有消息广播给所有消费者。我们希望扩展这一点，允许根据消息的严重性过滤消息。例如，我们可能希望将日志消息写入磁盘的脚本只接受严重错误，而不会在warning或info日志消息上浪费磁盘空间。

我们是有fanout交换器，这并没有给我们很大的灵活性--它只能无脑广播。

我们将使用direct交换器。direct交换器背后的路由算法很简单--消息进入其binding key与消息的routing key完全匹配的队列。

为了说明这一点，请考虑一下设置：

在此设置中，我们可以看见绑定了两个队列的direct交换器x。第一个队列绑定键为orange，第二个队列绑定为两个，一个绑定键为black，另一个为green。

在这种设置用，使用orange路由键发布到交换器的消息将被路由到队列Q1。路由键为black或green的消息将转到Q2。所有其他消息将被丢弃。

### 多重绑定

用相同的绑定键绑定多个队列是完全合法的。在我们的示例中，我们可以使用绑定键black在x和Q1之间添加绑定。在这种情况下，direct交换器将类似fanout，并将消息广播到所有匹配的队列。带有black路由键的消息同时传递给Q1和Q2，

### 发送日志

我们将日志系统中使用这个模型。我们将发送消息到direct交换器，而不是fanout。我们将提供严重性（通常我们将日志级别划分为日志信息的严重性）作为路由键。这样，接收脚本将能够选择其想接收的日志级别。让我们首先关注发送日志。

和往常一样，我们需要首先创建一个交换器

```go
err = ch.ExchangeDeclare*=(
    // name
    "logs_direct",
    // type
    "direct",
    // durable
    true,
    // auto_deleted
    false,
    // internal
    false,
    // no_wait
    false,
    // arguments
    nil,
)
```

我们已经准备好发送一条消息

```go
err = ch.ExchangeDeclare*=(
    // name
    "logs_direct",
    // type
    "direct",
    // durable
    true,
    // auto_deleted
    false,
    // internal
    false,
    // no_wait
    false,
    // arguments
    nil,
)

failOnError(err, "Failed to declare an exchange")

body := bodyFrom(os.Args)
err = ch.Publish(
    "logs_direct",
    severityFrom(os.Args),
    false,
    false,
    amqp.Publish{
        ContentType: "text/plain",
        Body:   []byte(body)
    }
)
```

为了简化问题，我们假设“严重性”可以是"info", "warning", "error"之一。

### 订阅

接收消息的工作方式与上一教程一样，但有一个例外--我们将为感兴趣的每种严重性（日志级别）创建一个新的绑定。

```go
q, err := ch.QueueDeclare(
    // name
    "",
    // durable
    false,
    // delect when unused
    false,
    // exclusive
    true,
    // no_wait
    false,
    // arguments
    nil,
)

failOnError(err, "Failed to declare a queue")

if len(os.Args) < 2 {
    log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
    os.Exit(0)
}

// 建立多个绑定关系
for _, s := range os.Args[1:] {
    log.Printf("Binding queue %s to exchange %s with routing key %s", q.Name, "log_direct", s)
    err = ch.QueueBind(
        // queue name
        q.Name,
        // routing key
        s,
        // exchange
        "logs_direct",
        false,
        nil,
    )
    failOnError(err, "Failed to bind a queue")
}
```
