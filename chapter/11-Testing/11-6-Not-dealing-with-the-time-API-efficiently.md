## 11.6 测试不稳定，都是 time API 在捣乱

有些函数必须依赖时间 API，例如，检索当前时间。在这种情况下，非常容易出现编写脆弱单元测试，这些测试在某些时候可能会失败。在本节中，我们将通过一个具体示例并讨论可能的选项。目标不是涵盖所有用例和技术，而是提供有关使用 time API 编写更健壮的函数测试的指导。

假设一个应用程序接收到我们想要存储在内存缓存中的事件。我们将实现一个 `Cache` 结构来保存最近的事件。该结构将公开三个方法：

* 追加事件
* 获取所有事件
* 修剪给定持续时间的事件（我们将关注此方法）

这些方法中的每一个都需要访问当前时间。让我们写一个使用 `time.Now()` 实现后一种方法的第一个实现（我们将假设所有事件都按时间排序）：

```go
type Cache struct {
    mu     sync.RWMutex
    events []Event
}

type Event struct {
    Timestamp time.Time
    Data string
}

func (c *Cache) TrimOlderThan(since time.Duration) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    t := time.Now().Add(-since)
    for i := 0; i < len(c.events); i++ {
        if c.events[i].Timestamp.After(t) {
            c.events = c.events[i:]
            return
        }
    }
}
```

首先，我们计算一个 `t` 变量，它是当前时间减去提供的持续时间。然后，随着事件按时间排序，一旦我们到达时间在 `t` 之后的事件，我们就会更新内部 `events` 切片。

现在，我们如何测试这种方法？我们可以使用 `time.Now` 依赖当前时间来创建事件：

```go
func TestCache_TrimOlderThan(t *testing.T) {
    events := []Event{
        {Timestamp: time.Now().Add(-20 * time.Millisecond)},
        {Timestamp: time.Now().Add(-10 * time.Millisecond)},
        {Timestamp: time.Now().Add(10 * time.Millisecond)},
    }
    cache := &Cache{}
    cache.Add(events)
    cache.TrimOlderThan(15 * time.Millisecond)
    got := cache.GetAll()
    expected := 2
    if len(got) != expected {
        t.Fatalf("expected %d, got %d", expected, len(got))
    }
}
```

在这里，我们使用 `time.Now()` 将事件片段添加到缓存中，并添加或减去一些小的持续时间。然后，我们将这些事件修剪 15 毫秒，然后执行断言。

这种方法有一个主要缺点。如果执行测试的机器突然很忙，我们可能会修剪比预期更少的事件。仍然可以增加所提供的持续时间以降低测试失败的百分比；然而，这并不总是可能的。例如，如果时间戳字段是添加事件时生成的未导出字段怎么办？在这种情况下，不可能传递特定的时间戳，并且可能最终会在单元测试中增加 sleep 时长。

问题与 `TrimOlderThan` 的实现有关。当它调用 `time.Now()` 时，它使得实现健壮的单元测试变得更加困难。让我们讨论两种主要的方法，它们可以使我们的测试不那么脆弱。

第一种方法是使检索当前时间的方式成为 `Cache` 结构的依赖项。在生产中，我们会注入真正的实现，而在单元测试中，我们会传递一个存根。

要处理这种依赖关系，有不同的选项，例如接口或函数类型。在我们的例子中，由于我们只依赖一个方法（`time.Now()`），我们可以定义一个函数类型：

```go
type now func() time.Time

type Cache struct {
    mu     sync.RWMutex
    events []Event
    now    now
}
```

`now` 类型是一个返回 `time.Time` 函数的函数，我们可以这样 传递实际的 `time.Now` 函数：

```go
func NewCache() *Cache {
    return &Cache{
        events: make([]Event, 0),
        now:    time.Now,
    }
}
```

由于 `now` 依赖项仍未导出，因此外部客户端无法访问它。此外，在我们的单元测试中，我们可以通过基于预定义时间注入 `func() time.Time` 的假实现来创建 `Cache` 结构：

```go
func TestCache_TrimOlderThan(t *testing.T) {
    events := []Event{
        {Timestamp: parseTime(t, "2020-01-01T12:00:00.04Z")},
        {Timestamp: parseTime(t, "2020-01-01T12:00:00.05Z")},
        {Timestamp: parseTime(t, "2020-01-01T12:00:00.06Z")},
    }
    cache := &Cache{now: func() time.Time {
        return parseTime(t, "2020-01-01T12:00:00.06Z")
    }}
    cache.Add(events)
    cache.TrimOlderThan(15 * time.Millisecond)
    // ...
}

func parseTime(t *testing.T, timestamp string) time.Time {
    // ...
}
```

在创建新的 `Cache` 结构时，我们根据给定时间注入 `now` 依赖项。多亏了这种方法，测试现在很健壮。即使在最坏的条件下，该测试的结果也是确定性的。

> **Note** 除了使用字段，还可以通过全局变量来检索时间：
>
>`var now = time.Now`
>
> 一般来说，我们应该倾向于防止拥有这种可变的共享状态。在我们的例子中，这将导致至少一个具体的问题：测试将不再是孤立的，因为它们都依赖于一个共享变量。因此，例如，测试不能并行运行。如果 可能的话，我们应该将这些情况作为结构依赖的一部分来处理，从而促进测试隔离。

该解决方案也是可扩展的。例如，如果函数调用 `time.After` 怎么办？我们可以添加另一个 `after` 依赖项，或者创建一个接口来组合这两种方法：`Now` 和 `After`。然而，这种方法有一个主要缺点。例如，如果我们从外部包创建单元测试，则 `now` 依赖项不可用（我们将在 *不探索所有 Go 测试功能* 中探索它）

在这种情况下，另一种选择是可能的；除了将时间作为未导出的依赖项处理之外，我们还可以要求客户端提供当前时间：

```go
func (c *Cache) TrimOlderThan(now time.Time, since time.Duration) {
    // ...
}
```

然而，更进一步，我们还可以将两个函数参数合并到一个单独的 `time.Time` 中，它代表一个特定的时间点，直到我们想要修剪事件：

```go
func (c *Cache) TrimOlderThan(t time.Time) {
    // ...
}
```

因此，由调用者来计算这个时间点：

```go
cache.TrimOlderThan(time.Now().Add(time.Second))
```

这将是相同的测试方式：

```go
func TestCache_TrimOlderThan(t *testing.T) {
    // ...
    cache.TrimOlderThan(parseTime(t, "2020-01-01T12:00:00.06Z").
        Add(-15 * time.Millisecond))
    // ...
}
```

此选项可能是最简单的，因为它不需要创建其他类型和存根。

一般来说，我们应该谨慎测试使用 `time` API 的代码。它也可以为脆弱测试敞开大门。在本节中，我们已经看到了两种处理它的方法。一种选择是将 `time` 交互保留为我们可以在单元测试中伪造的依赖项的一部分。我们可以实现自己的实现或依赖外部库。另一种更简单但更受限制的选择是重新设计我们的 API 并要求客户端向我们提供我们需要的信息，例如当前时间。

现在让我们深入研究两个与测试相关的有用 Go 包：`httptest` 和 `iotest`。