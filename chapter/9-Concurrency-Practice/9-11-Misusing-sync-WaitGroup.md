## 9.11 使用 sync.WaitGroup 引入的坑

`sync.WaitGroup` 是一种等待 _n_ 个操作完成的机制；通常，我们使用它来等待 _n_ 个 goroutine 完成。让我们首先关注我们公共 API；然后，我们将看到一个非常频繁的错误导致非确定性行为。

可以使用 `sync.WaitGroup` 的零值创建等待组：

```go
wg := sync.WaitGroup{}
```

在内部，`sync.WaitGroup` 拥有一个默认初始化为零的内部计数器。我们可以使用 `Add(int)` 方法增加这个计数器，并使用 `Done()` 或 `Add` 减少它的负值。最后但并非最不重要的一点是，如果我们想等待计数器为零，我们必须使用将阻塞的 `Wait()` 方法。

> **Note** 计数器不能为负数；否则，goroutine 会引发 panic。

在下面的示例中，我们将初始化一个等待组，启动三个将自动更新计数器的 goroutine，然后等待它们完成。最后，我们要等待这三个 goroutine 打印出计数器的值（应该是 3）。你能猜出这段代码是否有问题吗？

```go
wg := sync.WaitGroup{}
var v uint64

for i := 0; i < 3; i++ {
    go func() {
        wg.Add(1)
        atomic.AddUint64(&v, 1)
        wg.Done()
    }()
}

wg.Wait()
fmt.Println(v)
```

如果我们运行这个例子，我们将得到一个不确定的值，它可以打印从 0 到 3 的任何值。此外，我们可以注意到，如果我们启用竞争标志，Go 甚至会捕获数据竞争。当我们使用 `sync/atomic` 包来更新 `v` 时，可能会发生什么呢？这段代码出了什么问题？

这里的问题是 `wg.Add(1)` 是在新创建的 goroutine 中调用的，而不是在父 goroutine 中。因此，不能保证我们已经向等待组表明我们希望在调用 `wg.Wait()` 之前等待三个 goroutine。

让我们看看代码打印 2 时的可能场景：

![](https://img.exciting.net.cn/67.png)

在这种情况下，主 goroutine 会启动三个 goroutine。然而，最后一个 goroutine 在前两个 goroutine 已经调用 `wg.Done()` 之后执行。因此，父 goroutine 已经解锁。因此，在这种情况下，当 main goroutine 读取 `v` 时，它等于 2。此外，数据竞争探测器可以检测到对 `v` 的不安全访问。

在处理 goroutine 时，重要的是要记住，如果没有同步，执行是不确定的。例如，以下代码可以打印 `ab` 或 `ba`：

```go
go func() {
    fmt.Print("a")
}()
go func() {
    fmt.Print("b")
}()
```

事实上，两个 goroutine 都可以分配给不同的线程，并且不能保证哪个线程将首先执行。

CPU 必须使用所谓的内存栅栏（也称为内存屏障）来确保顺序。Go 提供了不同的同步技术来实现内存栅栏，例如 `sync.WaitGroup`，因为它启用了 `wg.Add` 和 `wg.Wait` 之间的 happens-before 关系。

回到我们的示例，有两种可能的选项来解决我们的问题。在循环之前调用 `wg.Add` 3：

```go
wg := sync.WaitGroup{}
var v uint64

wg.Add(3)
for i := 0; i < 3; i++ {
    go func() {
        // ...
    }()
}
// ...
```

或者在启动子 goroutine 之前在每次循环迭代期间调用 `wg.Add`：

```go
wg := sync.WaitGroup{}
var v uint64

for i := 0; i < 3; i++ {
    wg.Add(1)
    go func() {
        // ...
    }()
}

// ...
```

两种解决方案都很好。如果我们最终要设置到等待组计数器的值是预先知道的，那么第一个解决方案可以防止我们不得不多次调用 `wg.Add`。但是，它需要确保在所有地方使用相同的计数以避免细微的错误。

让我们小心不要重现 Go 开发人员犯的这个常见错误。使用 `sync.WaitGroup` 时，必须在父 goroutine 中启动一个 goroutine 之前完成 `Add` 操作，而 `Done` 操作必须在 goroutine 中完成。

下一节将讨论同步包的另一个原语：`sync.Cond`。