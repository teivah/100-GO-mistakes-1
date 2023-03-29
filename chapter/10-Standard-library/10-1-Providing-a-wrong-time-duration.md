## 10.1 提供错误的时间周期(time Duration)

标准库提供了接受 `time.Duration` 的通用函数或方法。然而，由于 `time.Duration` 是 `int64` 类型的别名，该语言的新手可能会感到困惑并提供错误的时间周期。例如，具有 Java 或 JavaScript 背景的开发人员习惯于传递数字类型。

为了说明这个常见错误，让我们创建一个新的 `time.Ticker`，它将以固定的时间间隔每秒传递时钟的滴答声：

```go
ticker := time.NewTicker(1000)
for {
    select {
    case <-ticker.C:
        // Do something
    }
}
```

如果我们运行这段代码，我们会注意到不是每秒都发送刻度；它们每微秒交付一次。

事实上，由于 `time.Duration` 是基于 `int64` 类型的，所以前面的代码是正确的，因为 `1000` 是一个有效的 `int64`。然而，`time.Duration` 表示两个 *纳秒* 之间经过的时间。因此，我们为 `NewTicker` 提供了 1000 纳秒 = 1 微秒的时间周期。

这种错误并不少见。事实上，Java 或 JavaScript 等语言的标准库有时会要求开发人员提供以毫秒为单位的时间周期。

此外，如果我们想有意识地创建一个时间间隔为一微秒的 `time.Ticker`，我们不应该像我们所做的那样直接传递一个 `int64`。 我们应该确保始终使用 `time.Duration` API 以避免可能的混淆：

```go
ticker = time.NewTicker(time.Microsecond)
// Or
ticker = time.NewTicker(1000 * time.Nanosecond)
```

这个常见错误也可能不是本书中最复杂的错误。但是，具有其他语言背景的开发人员很容易陷入陷阱，认为 `time` 包的函数和方法需要毫秒。请记住始终使用 `time.Duration` API 并在时间单位旁边提供 `int64`。

现在，让我们讨论一下使用带有 `time.After` 的 `time` 包时的一个常见错误。