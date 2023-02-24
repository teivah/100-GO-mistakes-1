## 11.5 在单元测试中使用 sleep 引入的坑

脆弱测试是一种可能在不更改任何代码的情况下通过或失败的测试。脆弱测试被认为是测试中最大的障碍之一，因为它们的调试成本很高，并且会削弱我们对测试准确性的信心。在 Go 中，调用 `time.Sleep` 在测试中可能是可能出现问题的一个很明显的信号。事实上，测试并发代码是相当频繁的，例如，使用 sleep。让我们在本节中了解从测试中删除 sleep 以防止编写不稳定测试的具体技术。

我们将用一个函数来说明本节，该函数返回一个值并启动一个在后台执行作业的 goroutine。我们将调用一个函数来获取 `Foo` 结构的切片并返回第一个元素。同时，另一个 goroutine 将负责调用具有前 `n` 个 `Foo` 元素的 `Publish` 方法：

```go
type Handler struct {
    n         int
    publisher publisher
}

type publisher interface {
    Publish([]Foo)
}

func (h Handler) getBestFoo(someInputs int) Foo {
    foos := getFoos(someInputs)
    best := foos[0]

    go func() {
        if len(foos) > h.n {
            foos = foos[:h.n]
        }
        h.publisher.Publish(foos)
    }()

    return best
}
```

`Handler` 结构包含两个字段：`n` 字段和用于发布第一个 `n` `Foo` 结构的 `publisher` 依赖项。首先，我们得到一个 `Foo` 的切片，但在返回第一个之前，我们启动一个新的 goroutine，过滤 `foos` 切片并调用 `Publish`。

我们如何测试这个功能？编写断言响应的部分非常简单。但是，如果我们还想检查传递给 `Publish` 的内容怎么办？

首先，我们可以模拟 `publisher` 接口来记录调用 `Publish` 方法时传递的参数。然后，第一种方法可能是在检查记录的参数之前 sleep 几毫秒：

```go
type publisherMock struct {
    mu  sync.RWMutex
    got []Foo
}

func (p *publisherMock) Publish(got []Foo) {
    p.mu.Lock()
    defer p.mu.Unlock()
    p.got = got
}

func (p *publisherMock) Get() []Foo {
    p.mu.RLock()
    defer p.mu.RUnlock()
    return p.got
}

func TestGetBestFoo(t *testing.T) {
    mock := publisherMock{}
    h := Handler{
        publisher: &mock,
        n:         2,
    }

    foo := h.getBestFoo(42)
    // Check foo
    
    time.Sleep(10 * time.Millisecond)
    published := mock.Get()
    // Check published
}
```

首先，我们编写了一个模拟的 `publisher`，它将依赖互斥锁来保护对 `published` 字段的访问。在我们的单元测试中，我们调用 sleep 在检查传递给 `Publish` 的参数之前离开一些时间。

这个测试本质上是不稳定的。不能严格保证这里的 10 毫秒就足够了（在本例中，可能但不能保证）。

那么，有哪些方法可以改进这个单元测试呢？

第一个选项是使用重试定期断言给定条件。例如，我们可以编写一个函数，它接受一个断言作为参数和最大重试次数加上一个等待时间，该时间会定期调用以避免繁忙循环：

```go
func assert(t *testing.T, assertion func() bool,
    maxRetry int, waitTime time.Duration) {
    for i := 0; i < maxRetry; i++ {
        if assertion() {
            return
        }
        time.Sleep(waitTime)
    }
    t.Fail()
}	
```

此函数检查提供的断言并在一定次数的重试后失败。我们也在使用 `time.Sleep`，但我们可以在此代码中使用更短的持续时间。例如，回到 `TestGetBestFoo`：

```go
assert(t, func() bool {
    return len(mock.Get()) == 2 
}, 30, time.Millisecond)
```

我们不是 sleep 十毫秒，而是每毫秒 sleep 并配置最大重试次数。如果测试成功，这种方法会减少执行时间，因为这意味着减少等待间隔。因此，实施重试策略是比使用被动 sleep 更好的方法。

> **Note** 一些测试库（例如 testify）也提供重试功能。例如，在 `testify` 中，我们可以使用 `Ultimate` 函数来实现所描述的内容以及其他功能，例如配置错误消息。

另一种策略可能是使用通道来同步 goroutine，发布 `Foo` 结构和测试 goroutine。例如，在模拟实现中，我们可以将这个值发送到一个通道，而不是将接收到的切片复制到一个字段中：

```go
type publisherMock struct {
    ch chan []Foo
}

func (p *publisherMock) Publish(got []Foo) {
    p.ch <- got
}

func TestGetBestFoo(t *testing.T) {
    mock := publisherMock{
        ch: make(chan []Foo),
    }
    defer close(mock.ch)

    h := Handler{
        publisher: &mock,
        n:         2,
    }
    foo := h.getBestFoo(42)
    // Check foo

    if v := len(<-mock.ch); v != 2 {
        t.Fatalf("expected 2, got %d", v)
    }
}
```

发布者将收到的参数发送到通道。同时，测试 goroutine 设置模拟并根据接收到的值创建断言。也可以实施超时策略，以确保我们不会在发生错误时永远等待 `mock.ch`。例如，我们可以使用带有 `time.After` 案例的 `select` 来做到这一点。

我们应该支持哪个选项：重试还是同步？通常，如果可以进行同步，则它可能是默认选择。事实上，如果设计得当，它将等待时间减少到最低限度，并使其具有完全确定性。如果我们不能应用同步，我们或许应该重新考虑我们的设计，因为我们可能有问题。如果同步确实不可能，我们应该使用重试选项，无论如何，这将是比被动 sleep 更好的选择，以消除测试中的不确定性。

让我们继续讨论如何防止在测试中出现不稳定，这次是使用 time API。