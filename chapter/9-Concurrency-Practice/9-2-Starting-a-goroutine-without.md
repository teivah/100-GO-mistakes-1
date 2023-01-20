## 9.2 goroutine 已启动，啥时候停，不确定

Goroutines 上手成本低。如此轻松加愉快，我们一开始可能没有计划关于何时停止 goroutine，这可能导致泄漏。不知道何时停止 goroutine 是一个设计问题，也是 Go 中常见的并发错误。让我们了解为什么以及如何防止它。

首先，让我们量化一下 goroutine 泄漏意味着什么。在内存方面，goroutine 以最小 2KB 的堆栈大小开始，可以根据需要增长和缩小（64 位的最大堆栈大小为 1 GB，32 位的最大堆栈 大小为 250 MB）。在内存方面，goroutine 还可以保存分配给堆的变量引用。同时，goroutine 可以保存资源，例如 HTTP 或 DB 连接、打开的文件、最终应该正常关闭的网络套接字。如果一个 goroutine 被泄露，这些资源也会被泄露。

让我们看一个 goroutine 的不确定停止的例子。在这里，父 goroutine 调用一个返回通道的函数，然后创建一个新的 goroutine 将继续从该通道接收消息：

```go
ch := foo()
go func() {
    for v := range ch {
        // ...
    }
}()
```

创建的 goroutine 将在 `ch` 关闭时退出。然而，我们是否确切知道该通道何时关闭？可能不确定，因为 `ch` 是由 `foo` 函数创建的。如果通道从未关闭，那么它就是泄漏。因此，我们应该始终对 
goroutine 的退出点保持谨慎，并确保最终到达其中一个。

现在，让我们讨论一个具体的例子。我们将设计一个需要监视一些外部配置的应用程序（例如，使用数据库连接）。这是第一个实现：

```go
func main() {
    newWatcher()

    // Run the application
}

type watcher struct { /* Some resources */ }

func newWatcher() {
    w := watcher{}
    go w.watch()
}
```

我们调用 `newWatcher`，它创建一个 `watcher` 结构并启动一个 goroutine 负责监视配置。这段代码的问题在于，当主 goroutine 退出时（可能是因为操作系统信号或因为它的工作量有限），应用程序将停止。因此，`watcher` 创建的资源不会被正常关闭。我们怎样才能防止这种情况发生？

一种选择是向 `newWatcher` 传递一个上下文，该上下文将在 `main` 返回时被取消：

```go
func main() {
    ctx, cancel :=
    context.WithCancel(context.Background())
    defer cancel()

    newWatcher(ctx)

    // Run the application
}

func newWatcher(ctx context.Context) {
    w := watcher{}
    go w.watch(ctx)
}
```

我们将创建的上下文传到 `watch` 方法。当上下文被取消时，`watcher` 结构应该关闭它的资源。但是，我们能保证 `watch` 有时间这样做吗？绝对没有，这是一个设计缺陷。

这里的问题是我们使用信号来传达一个 goroutine 必须停止。在资源关闭之前，我们没有阻塞父 goroutine。现在让我们确保我们这样做：

```go
func main() {
    w := newWatcher()
    defer w.close()

    // Run the application
}

func newWatcher() watcher {
    w := watcher{}
    go w.watch()
    return w
}

func (w watcher) close() {
    // Close the resources
}
```

`watcher` 有一个新方法：`close`。我们现在使用 defer 来调用这个 `close` 方法，以保证资源在应用程序退出之前关闭，而不是向观察者发出是时候关闭其资源的信号。

总而言之，让我们注意 goroutine 是一种资源，就像任何其他资源一样，最终必须关闭，无论是释放内存还是其他资源。在不知道何时停止的情况下启动 goroutine 是一个设计问题。每当一个 goroutine 启动时，我们都应该对它何时停止有一个明确的计划。最后但同样重 要的是，如果一个 goroutine 创建资源并且它的生命周期与应用程序的生命周期绑定，那么等待它关闭而不是通知它可能更安全。这样，我们可以确保在退出应用程序之前释放资源。

现在让我们深入研究在 Go 中工作时最常见的错误之一：goroutines 和循环变量。