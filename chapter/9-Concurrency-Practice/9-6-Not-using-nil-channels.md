## 9.6 不要使用零值通道

在 Go 和通道中工作时很常见的一个错误是 nil 通道有可能会引入一些坑。那么什么是 nil 通道，我们为什么要关心它们呢？

让我们从一个 goroutine 开始，它将创建一个 nil 通道并等待接收消息。这段代码应该做什么？

```go
var ch chan int
<-ch
```

`ch` 是 `chan int` 类型。通道的零值为 nil，`ch` 为 nil。在 nil 通道上接收消息是有效的操作。goroutine 不会触发 panic；但是，它将永远阻塞。

如果我们向 nil 通道发送消息，这也是同样的原理。这个 goroutine 也将永远阻塞：

```go
var ch chan int ch <‑ 0
```

那么，允许接收或发送消息到 nil 通道的目的是什么？我们将通过一个具体的例子来讨论这个问题。

我们将实现一个 `func merge(ch1, ch2 <-chan int) <-chan int` 函数将两个通道合并为一个通道。通过合并它们，我们的意思是在 `ch1` 或 `ch2` 中接收到的每条消息都将发送到返回的通道：

![](https://img.exciting.net.cn/63.png)

那么我们如何在 Go 中做到这一点呢？让我们首先编写一个简单的实现，它启动一个 goroutine 并从两个通道接收（结果通道将是一个元素的缓冲通道）：

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    ch := make(chan int, 1)

    go func() {
        for v := range ch1 {
            ch <- v
        }
        for v := range ch2 {
            ch <- v
        }
        close(ch)
    }()

    return ch
}
```

在另一个 goroutine 中，我们从两个通道接收，并且每条消息最终都在 `ch` 中发布。

第一个版本的主要问题是我们先从 `ch1` 接收，*然后* 再从 `ch2` 接收。这意味着在 `ch1` 关闭之前我们不会从 `ch2` 接收到数据。假设它不适合我们的用例，因为 `ch1` 可能永远打开，所以我们想同时从两个通道接收。

让我们使用 `select` 编写一个带有并发接收器的改进版本：

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    ch := make(chan int, 1)

    go func() {
        for {
            select {
            case v := <-ch1:
                ch1:
                ch <- v
            case v := <-ch2:
                ch <- v
            }
        }
        close(ch)
    }()
	
    return ch
}
```

`select` 语句让 goroutine 同时等待多个操作。当我们将它包装在一个 `for` 循环中时，我们应该反复从一个或另一个通道接收消息，对吗？但是这段代码真的有效吗？

一个问题是无法访问 `close(ch)` 语句。实际上，当通道关闭时，使用 `range` 运算符在通道上循环会中断。但是，当 `ch1` 或 `ch2` 关闭时，我们实现 `for / select` 的方式不会捕获。更糟糕的是，如果在某个时候 `ch1` 或 `ch2` 关闭，合并通道的接收者在记录值时会收到以下内容：

这是合并通道的接收者在记录值时将收到的内容：

```shell
received: 0
received: 0
received: 0
received: 0
received: 0
...
```

所以接收者会重复接收一个等于 0 的整数。是什么原因？从关闭的通道接收是非阻塞操作：

```go
ch1 := make(chan int)
close(ch1)
fmt.Print(<-ch1, <-ch1)
```

尽管人们可能认为这段代码会崩溃或阻塞，但这段代码会运行并打印 `0 0`。实际上，我们在这里捕获的是关闭事件，而不是实际的消息。要检查我们是否收到消息或关闭信号，我们必须这样做：

```go
ch1 := make(chan int)
close(ch1)
v, open := <-ch1
fmt.Print(v, open)
```

使用 `open` 布尔值，我们现在可以注意到 `ch1` 是否仍然打开：

```shell
false
```

同时，我们也将 0 分配给 `v`，因为它是整数的零值。

让我们回到我们的第二个解决方案。我们说过，如果 `ch1` 关闭，它不会很好地工作；例如，因为 `select case` 是 `case v := <-ch1`，我们将继续输入这个 case 并向合并通道发布一个零整数。

让我们退后一步，看看解决这个问题的最佳方法是什么：

![](https://img.exciting.net.cn/64.png)

我们必须从两个渠道接收。

* `ch1` 首先关闭，所以我们必须从 `ch2` 接收直到它关闭
* `ch2` 首先关闭，所以我们必须从 `ch1` 接收直到它关闭

我们如何在 Go 中实现这一点？让我们尝试编写一个可能已经使用状态机方法和布尔值完成的版本：

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    ch := make(chan int, 1)
    ch1Closed := false
    ch2Closed := false

    go func() {
        for {
            select {
            case v, open := <-ch1:
                if !open {
                    ch1Closed = true
                    break
                }
                ch <- v
            case v, open := <-ch2:
                if !open {
                    ch2Closed = true
                    break
                }
                ch <- v
            }

            if ch1Closed && ch2Closed {
                close(ch)
                return
            }
        }
    }()

    return ch
}
```

我们定义了两个布尔值 `ch1Closed` 和 `ch2Closed`。一旦我们收到来自通道的消息，我们就会检查它是否是一个关闭信号。如果是，我们通过将通道标记为关闭（例如 `ch1Closed = true`）来处理它。一旦两个通道都关闭，我们关闭合并的通道并停止 goroutine。

除了开始变得复杂之外，该代码还有什么问题？有一个主要问题：当两个通道之一关闭时，`for` 循环将充当忙等待循环，这意味着即使另一个通道没有收到新消息，它也会继续循环。事实上，我们必须牢记示例中 `select` 语句的行为。假设 `ch1` 已关闭（因此我们不会在此处收到任何新消息）；一旦我们再次到达 `select`，它将等待这三个条件之一发生：

* `ch1` 关闭
* `ch2` 有一条新消息
* `ch2` 关闭

第一个条件 `ch1` 关闭，将始终有效。因此，只要我们没有在 `ch2` 中收到消息或者它已关闭，我们就会继续循环第一种情况。这将导致浪费 CPU 周期，应该避免。因此，我们的解决方案是不可行的。

我们可以尝试增强状态机部分并在每种情况下实现次级 `for/select` 循环。但这会使我们的代码更加复杂和难以理解。

现在是回到零值通道的正确时机。正如我们所提到的，从零通道接收将永远阻塞。在我们的解决方案中利用这个想法怎么样？一旦通道关闭，我们不会设置布尔值，而是将此通道分配给 nil。让我们编写最终版本：

```go
func merge(ch1, ch2 <-chan int) <-chan int {
    ch := make(chan int, 1)

    go func() {
        for ch1 != nil || ch2 != nil {
            select {
            case v, open := <-ch1:
                if !open {
                    ch1 = nil
                    break
                }
                ch <- v
            case v, open := <-ch2:
                if !open {
                    ch2 = nil
                    break
                }
                ch <- v
            }
        }
        close(ch)
    }()

    return ch
}
```

首先，只要至少有一个通道仍然打开，我们就会循环。然后，例如，如果 `ch1` 关闭，我们将 `ch1` 分配给 nil。因此，在下一次循环迭代期间，`select` 语句现在将只等待两个条件：

* `ch2` 有一条新消息
* `ch2` 关闭

`ch1` 甚至不再处于平衡状态，因为它是一个零通道。同时，我们为 `ch2` 保留相同的逻辑，并在关闭后将其分配给 nil。最后，当两个通道都关闭时，我们关闭合并的通道并返回。这是此实现的模型：

![](https://img.exciting.net.cn/65.png)

这是我们一直在等待的实现。我们涵盖了所有不同的情况，它不需要会浪费 CPU 周期的繁忙循环。

总而言之，我们已经看到等待或发送到 nil 通道是一种阻塞操作，这种行为根本不是无用的。正如我们在整个合并两个通道的示例中所看到的那样，我们可以使用 nil 通道来实现一个优雅的状态机，该状态机将从 `select` 语句中删除一个案例。让我们记住这个想法：nil 通道在某些情况下确实有用，并且在处理并发代码时应该成为 Go 开发人员工具集的一部分。

在下一节中，我们将讨论创建通道时应设置的大小。