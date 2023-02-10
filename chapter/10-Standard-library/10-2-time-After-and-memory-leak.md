## 10.2 time.After 引发的内存泄漏

`time.After(time.Duration)` 是一个方便的函数，它等待提供的持续时间耗尽，然后向返回的通道发送消息。通常，它用在并发代码中被使用；否则，如果我们想暂停一段指定的时间，我们可以使用 `time.Sleep(time.Duration)`。`time.After` 的优势在于可以用来实现“如果我在这个通道 5 秒内没有收到任何消息，那么我将……”之类的场景。然而，在代码库中经常发现对 `time.After` 的调用，正如我们将在本节中描述的那样，这可能是内存泄漏的根本原因。

让我们考虑以下示例。我们将实现一个重复使用来自通道的消息的函数。同时，如果我们超过一个小时没有收到任何消息，我们想记录一个警告。这是一个可能的实现：

```go
func consumer(ch <-chan Event) {
    for {
        select {
        case event := <-ch:
            handle(event)
        case <-time.After(time.Hour):
            log.Println("warning: no messages received")
        }
    }
}
```

在这里，我们在两种情况下使用 `select`：从 `ch` 接收消息或在一小时后没有消息（因为 `time.After` 在每次迭代期间评估，每次都重置超时）。乍一看，这段代码看起来没问题。但是，它可能会导致内存使用问题。

正如我们所说，`time.After` 返回一个通道。我们可能希望在每次循环迭代期间关闭此通道；然而，事实并非如此。`time.After` 创建的资源（包括通道）将在超时到期后释放，并会使用一些内存直到它发生。多少内存？在 Go 1.15 中，每次调用 `time.After` 大约需要 200 字节的内存。如果我们收到大量的消息，比如每小时 500 万条，我们的应用程序将消耗 1GB 的内存来存储时间资源。

我们可以通过在每次迭代期间以编程方式关闭通道来解决这个问题吗？这是不可能的。返回的通道是一个 `<-chan time.Time` 意味着一个不能关闭的只接收通道。

如果我们想修复我们的示例，有不同的选择。

第一个选项可能是使用上下文而不是 `time.After`：

```go
func consumer(ch <-chan Event) {
    for {
        ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
        select {
        case event := <-ch:
            cancel()
            handle(event)
        case <-ctx.Done():
            log.Println("warning: no messages received")
        }
    }
}
```

这种方法的缺点是我们必须在每次循环迭代期间不断重新创建上下文。创建上下文并不是 Go 中最轻量级的操作，因为它需要创建一个通。我们能做得更好吗？

第二个选项再次来自 `time` 包：`time.NewTimer` 此函数创建一个 `time.Timer` 结构，该结构导出：

* 一个 `C` 字段，它是内部定时器通道
* 重置持续时间的 `Reset(time.Duration)` 方法
* 停止计时器的 `Stop()` 方法

> **Note** 需要注意的是 `time.After` 也依赖于 `time.Timer`。但是，它只返回 `C` 字段。因此，我们无权访问 `Reset` 方法：

```go
package time

func After(d Duration) <-chan Time {
    return NewTimer(d).C
}
```

让我们使用 `time.NewTimer` 实现一个新版本：

```go
func consumer(ch <-chan Event) {
    timerDuration := 1 * time.Hour
    timer := time.NewTimer(timerDuration)

    for {
        timer.Reset(timerDuration)
        select {
        case event := <-ch:
            handle(event)
        case <-timer.C:
            log.Println("warning: no messages received")
        }
    }
}
```

在这个实现中，我们在每次循环迭代期间保持一个重复的动作：调用 `Reset` 方法。但是，调用 `Reset` 比每次都必须创建一个新的上下文更简单。它更快并且对 GC 的压力更小，因为它不需要任何新的堆分配。因此，使用 `time.Timer` 是我们最初问题的最佳解决方案。

> **Note** 为了简单起见，在前面的例子中，前面的 goroutine 并没有停止。正如我们在不知道 _何时停止的情况下启动 goroutine_ 中提到的，这不是最佳实践。在生产级代码中，我们应该找到一个退出条件，例如取消的上下文。在这种情况下，我们还应该记住停止使用 `defer timer.Stop()` 创建的 `time.Timer`，例如在创建 `timer` 之后立即停止。

在循环中使用 `time.After` 并不是唯一可能导致内存消耗达到峰值的情况。该问题与重复调用的代码有关。循环是一种情况，但在 HTTP 处理函数中使用 `time.After` 也会导致相同的问题，因为该函数将被多次调用。

一般来说，我们在使用 `time.After` 时要谨慎。请记住，创建的资源只有在计时器到期时才会被释放。当对 `time.After` 的调用被重复时（例如，在一个循环中，一个 Kafka 消费者函数，一个 HTTP 处理程序），它可能会导致内存消耗的峰值。在这种情况下，我们应该使用 `time.NewTimer`。

以下部分将深入探讨 JSON 处理中最常见的错误。