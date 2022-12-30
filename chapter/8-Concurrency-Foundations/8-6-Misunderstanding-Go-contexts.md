## 8.6 令人头大的 Go 上下文

开发人员有时会误解 `context.Context` 类型，尽管它是语言的关键概念之一，也是 Go 中并发代码的基础之一。让我们深入研究这个概念，并确保理解为什么以及如何有效地使用它。

官方文档：

一个上下文携带一个截止日期、一个取消信号和其他跨 API 边界的值。

让我们深入研究这个定义并理解与 Go 上下文相关的所有概念。

### 8.6.1 截止日期

截止日期是指通过以下方式确定的特定时间点：

* 一个 `time.Duration` 类型的属性，从当前开始计时（例如，250毫秒）
* 一个 `time.Time` 类型的属性，（例如，2023-02-07 00:00:00）

截止日期的语义传达了如果超过此截止日期，则应该停止正在进行的活动。例如，一个活动是一个 I/O 请求，或者一个等待从通道接收消息的 goroutine。

例如，让我们考虑一个每四秒从雷达接收一次飞行位置的应用程序。一旦我们收到一个位置，我们想与其他只对最新职位感兴趣的应用程序共享它。我们有一个包含单一方法的发布者接口：

```go
type publisher interface {
    Publish(ctx context.Context, position flight.Position) error
}
```

此方法接受上下文和位置。我们将假设具体实现将调用一个函数来向代理发布消息（例如，使用 Sarama 发布 Kafka 消息）。这个函数将是上下文感知的，这意味着一旦上下文被取消，它就可以取消请求。

假设我们没有收到现有的上下文，我们应该为 `Publish` 方法提供什么上下文参数？

我们已经提到所有应用程序只对最新位置感兴趣。因此，我们将构建的上下文应该传达出，在四秒后，如果我们无法发布飞行位置，我们应该停止它：

```go
type publishHandler struct {
    pub publisher
}

func (h publishHandler) publishPosition(position flight.Position) error {
    ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
    defer cancel()
    return h.pub.Publish(ctx, position)
}
```

此代码使用 `context.WithTimeout` 函数创建一个上下文。此函数接受超时和上下文。在这里，由于 `publishPosition` 没有接收到现有的上下文，我们使用 `context.Background` 从一个空的上下文创建一个。同时，`context.WithTimeout` 返回两个变量，创建的上下文和一个取消 `func()` 函数，该函数将在调用后取消上下文。将创建的上下文传递给 `Publish` 方法应该使其最多在四秒内返回。

另外，将 `cancel` 函数调用为延迟函数的理由是什么？在内部，`context.WithTimeout` 创建一个 goroutine，它将在内存中保留四秒钟或直到调用取消。因此，将 `cancel` 作为 defer 函数调用，意味着当我们退出父函数时，context 会被 cancel 掉，创建的 goroutine 也会停止。将保留的对象留在内存中是一种不返回的保护措施。

现在让我们转到 Go 上下文的第二个方面：取消信号。

### 8.6.2 取消信号

Go 上下文的另一个用例是携带取消信号。

假设我们想要创建一个在另一个 goroutine 中调用 `CreateFileWatcher(ctx context.Context, filename string)` 的应用程序。此函数将创建一个特定的文件观察程序，它将继续从文件中读取并捕获更新。当提供的上下文过期或被取消时，此函数处理它以关闭文件描述符。

最后但同样重要的是，当 main 返回时，我们希望通过关闭此文件描述符来优雅地处理事情。因此，我们需要传递一个信号。

一种可能的方法是使用 `context.WithCancel`，它返回一个上下文（第一个返回变量），一旦调用取消函数（第二个返回变量）就会取消：

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    go func () {
        CreateFileWatcher(ctx, "foo.txt")
    }()
}
```

当 main 函数返回时，它会调用 `cancel` 函数取消传递给 `CreateFileWatcher` 的上下文，从而优雅地关闭文件描述符。

现在让我们深入研究 Go 上下文的最后一个方面：值。

### 8.6.3 上下文的值

Go 上下文的最后一个用例是携带一个键/值列表。了解它背后的原理之前，让我们先看看如何使用它。

可以通过以下方式创建传达值的上下文：

```go
ctx := context.WithValue(parentCtx, "key", "value")
```

就像 `context.WithTimeout`,`context.WithDeadline` 和
 `context.WithCancel`,`context.WithValue` 是由父上下文创建的（这里是 `parentCtx`）。在这里，我们创建了一个新的 `ctx` 上下文，它包含与 `parentCtx` 相同的特征，但也传达了一个键和一个值。

我们可以使用 `Value` 方法访问该值：

```go
ctx := context.WithValue(context.Background(), "key", "value") 
fmt.Println(ctx.Value("key"))
```

```shell
value
```

提供的键和值是任何类型。对于值，我们想传递任何类型。但是，为什么键也应该是一个空接口而不是一个字符串？在这种情况下，它可能会导致碰撞。实际上，来自不同包的两个函数可以使用相同的字符串值作为键。因此，后者将覆盖前一个值。因此，处理上下文键时的最佳做法是创建一个未导出的自定义类型：

```go
package provider
    
type key string

const myCustomKey key = "key"

func f(ctx context.Context) {
    ctx = context.WithValue(ctx, myCustomKey, "foo")
    // ...
}
```

此处，未导出 `myCustomKey` 常量。因此，不存在使用相同上下文的另一个包覆盖已设置值的风险。即使另一个包也基于 key 类型创建了相同的 `myCustomKey`，它也将是一个不同的 key。

那么让上下文携带一个键/值列表有什么意义呢？由于 Go 上下文是通用的和主流的，因此有无限的用例。

例如，如果我们使用跟踪，我们可能希望不同的子函数共享相同的关联 ID。人们可能会认为 这个 ID 太具有侵入性而不能成为函数定义的一部分。在这方面，我们也可以决定将其作为提供的上下文的一部分。

另一个例子是如果我们想实现一个 HTTP 中间件。如果您不熟悉这样的概念，中间件是在服务请求之前执行的中间函数：

![](https://img.exciting.net.cn/56.png)

在此示例中，我们配置了两个中间件，必须在执行处理程序本身之前执行它们。如果我们希望中间件一起通信，它必须通过 `*http.Request` 处理的上下文。

让我们写一个中间件的例子来标记源主机是否是有效的：

```go
type key string
    
const isValidHostKey key = "isValidHost"

func checkValid(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        validHost := r.Host == "acme"
        ctx := context.WithValue(r.Context(), isValidHostKey, validHost)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

首先，我们定义一个名为 `isValidHostKey` 的特定上下文键。然后，`checkValid` 中间件检查源主机是否有效。此信息将在新的上下文中传达，使用 `next.ServeHTTP` 传递到下一个 HTTP 步骤（下一步可以是另一个 HTTP 中间件或最终的 HTTP 处理程序）。

这是一个示例，展示了如何在具体的 Go 应用程序中使用带值的上下文。

我们在以上的部分中已经看到如何创建一个上下文来携带截止日期、取消信号和/或值。我们可以使用这个上下文并将其传递给 *上下文感知* 库，这意味着库公开接受上下文的函数。但是现在，假设我们在另一个角度；我们现在得创建一个库，并且我们希望外部客户端提供可以取消的上下文。

### 8.6.4 捕获上下文取消事件

`context.Context` 类型导出一个 `Done` 方法，该方法返回一个只接收通知通道：`←chan struct{}`。当应取消与上下文关联的工作时，此通道将关闭。例如，与使用以下内容创建的上下文相关的 `Done` 通道：

* `context.WithCancel` 在取消函数调用时，被关闭。
* `context.WithDeadline` 在超过截止日期后，被关闭。

需要提一提的是，当上下文被取消或已达到截止日期，而不是接收特定值时，为什么要关闭内部通道？因为通道的关闭是所有消费者 goroutine 将收到的唯一通道动作。这样，一旦上下文被取消或达到截止日期，所有消费者都会收到通知。

此外，`context.Context` 导出一个 `Err` 方法，如果 `Done` 通道尚未关闭，该方法返回 nil； 否则，它会返回一个非零错误来解释 `Done` 通道关闭的原因，例如：

* 如果 channel 被取消，则会出现 `context.Canceled` 错误.
* 如果超过上下文的截止日期，则会出现 `content.DeadlineExceeded` 错误。

让我们看一个具体的例子，我们希望继续从频道接收消息。同时，我们的实现应该是上下文感知的，并在提供的上下文完成时返回：

```go
func handler(ctx context.Context, ch chan Message) error {
     for {
         select {
         case msg := <-ch:
                 // Do something with msg
         case <-ctx.Done():
                 return ctx.Err()
         }
     }
}
```

我们创建了一个 for 循环，并在两种情况下使用 `select`：从 `ch` 接收消息或接收上下文已完成且我们必须停止工作的信号。在处理通道时，这是一个如何使函数具有上下文感知的示例。

> **Note** 在接收传达可能取消或超时的上下文的函数中，接收或发送消息到通道的操作不应以阻塞方式完成。例如，在下面的函数中，我们将从一个频道接收信息并向另一个频道发送消息：

```go
func f(ctx context.Context) error {
        // ...
        ch1 <- struct{}{}

        v := <-ch2
        // ...
}
```

> 这个函数的问题是，如果上下文被取消或超时，我们可能不得不等 待消息发送或接收。相反，我们应该使用 `select` 来等待通道操作完成或等待上下文取消：

```go
func f(ctx context.Context) error {
        // ...
        select {
        case <-ctx.Done():
                return ctx.Err()
        case ch1 <- struct{}{}:
        }

        select {
        case <-ctx.Done():
                return ctx.Err()
        case v := <-ch2:
                // ...
        }
}
```

总之，如果我们想成为一名熟练的 Go 开发人员，我们必须了解上下文是什么以及如何使用它。事实上，在 Go 中，`context.Context` 无处不在，无论是在标准库还是外部库中。正如我们提到的，上下文允许携带截止日期、取消信号和/或键/值列表。一般来说，用户等待的函数应该使用上下文，因为它允许上游调用者决定何时中止它。

当不确定要使用哪个上下文时，我们应该使用 `context.TODO()` 而不是使用 `context.Background` 传递一个空的上下文。实际上，`context.TODO()` 返回一个空的上下文，但在语义上，它传达了要使用的上下文要么不清楚，要么还不可用（例如，因为还没有被父级传播）。

最后但同样重要的是，让我们注意标准库中的可用上下文对于多个 goroutine 并发使用都是安全的。