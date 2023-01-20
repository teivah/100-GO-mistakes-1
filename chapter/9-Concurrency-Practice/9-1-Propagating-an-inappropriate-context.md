# 9 并发：实践

> **本章概要**
> * 防止关于主要 Go 并发原语的常见错误：goroutines 和 channels
> * 了解使用标准数据结构和并发代码的影响
> * 深入研究标准库和一些扩展
> * 讨论可能导致数据竞争甚至死锁情况的细微错误

在上一章中，我们讨论了有关并发的基础。现在，是时候深入研究 Go 开发人员在使用并发原语时所犯的具体错误了。

## 9.1 传递不恰当的上下文

在 Go 中处理并发时，上下文无处不在，在许多情况下，可能建议传递它们。但是，有时上下文传递会导致细微的错误，从而阻止子功能正确执行。

让我们考虑以下示例。我们公开了一个执行某些任务并返回响应的 HTTP 处理程序。然而，就在返回响应之前，我们还想将其发送到 Kafka 的 topic。由于我们不想增加 HTTP 消费者的响应时长，我们希望在新的 goroutine 中异步处理发布操作。假设我们有一个 `publish` 函数供我们使用，它接受一个上下文，这样如果上下文被取消，发布消息的操作就可以被中断，例如。

这将是一个可能的实现：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    response, err := doSomeTask(r.Context(), r)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    go func() {
        err := publish(r.Context(), response)
        // Do something with err
    }()

    writeResponse(response)
}
```

首先，我们调用 `doSomeTask` 函数来获取 `response` 变量。它在 goroutine 调用 `publish` 和格式化 HTTP 响应中使用。此外，在调用 `publish` 时，我们会传递附加到 HTTP 请求的上下文。你能猜出这段代码有什么问题吗？

我们必须知道附加到 HTTP 请求的上下文可以在哪些的条件下取消：

* 当客户端连接关闭时
* 在 HTTP/2 请求下，当请求被取消时
* 或者当响应被写回客户端时

在前两种情况下，我们可能会正确处理事情。例如，如果我们刚刚收到来自 `doSomeTask` 的响应，但客户端已关闭连接，则可以在上下文已取消的情况下调用 `publish`，这样消息就不会发布。但是最后一种情况呢？

当响应被写入客户端时，与请求关联的上下文将被取消。因此，我们面临一个竞争条件：

* 如果写响应是在 Kafka 发布之后完成的，我们都会返回响应并成功发布消息
* 但是，如果在 Kafka 发布之前或期间写回响应，则消息可能不会发布

在后一种情况下，调用 `publish` 会返回错误；只是因为我们很快返回了 HTTP 响应。

那么我们该如何解决这个问题呢？一种想法可能是不传递父上下文。相反，我们会使用空上下文调用 `publish`：

```go
err := publish(context.Background(), response)
```

在这里，那会奏效。无论写回 HTTP 响应需要多长时间，我们都可以调用 `publish`。

然而，如果上下文包含一些有用的值怎么办？例如，如果上下文包含用于分布式跟踪的关联 ID，我们可以关联 HTTP 请求和 Kafka 发布。理想情况下，我们希望有一个新的上下文，与潜在的父取消分离，但仍然传达值。

标准包没有立即解决这个问题。因此，一个可能的解决方案是实现我们自己的 Go 上下文，类似于提供的上下文，只是它不携带取消信号。

`context.Context` 是一个包含 x 个方法的接口：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

上下文的截止日期由 `Deadline` 方法管理，取消信号通过 `Done` 和 `Err` 方法管理。当截止日期已过或上下文已被取消时，`Done` 应该返回一个关闭的通道，而 `Err` 应该返回一个错误。最后，这些值是通过 `Value` 方法携带的。

让我们创建一个自定义上下文，它将取消信号从父上下文中分离出来：

```go
type detach struct {
    ctx context.Context
}

func (d detach) Deadline() (time.Time, bool) {
    return time.Time{}, false
}

func (d detach) Done() <-chan struct{} {
    return nil
}

func (d detach) Err() error {
    return nil
}

func (d detach) Value(key any) any {
    return d.ctx.Value(key)
}
```

除了调用父上下文获取值的 `Value` 方法外，其他方法都返回一个默认值，因此上下文永远不会被视为过期或取消。

感谢我们的自定义上下文，我们现在可以调用 `publish` 和分离取消信号：

```go
err := publish(detach{ctx: r.Context()}, response)
```

现在，传递给 `publish` 的上下文是一个永远不会过期也不会被取消的上下文，但它会携带父上下文的值。

总之，传递上下文应该谨慎进行。我们使用基于与 HTTP 请求关联的上下文处理异步操作的示例来说明本节。由于一旦我们返回响应上下文就会被取消，异步操作也可能会意外停止。让我们牢记传递给定上下文的影响，如果有必要，我们还要记住，始终可以为特定操作创建自定义上下文。

下一节将讨论一个常见的并发错误：启动一个 goroutine 而没有计划停止它。