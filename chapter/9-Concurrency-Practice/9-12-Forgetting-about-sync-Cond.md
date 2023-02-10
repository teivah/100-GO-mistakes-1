## 9.12 被遗忘的 sync.Cond

在 `sync` 包中的同步原语中，`sync.Cond` 可能是最少被使用和理解的。但是，它带来了我们通过通道无法实现的功能。本节将通过一个具体示例来了解 `sync.Cond` 何时有用以及如何使用它。

本节的示例旨在实现捐赠目标机制，这意味着应用程序会在达到特定目标时发出警报。

我们创建一个 goroutine 负责增加余额（我们称之为 updater goroutine）。相对而言，其他 goroutine 将接收更新并在达到特定目标时打印一条消息（我们称其为 listener goroutines）。例如，一个 goroutine 正在等待 10 美元的捐赠目标，而另一个 goroutine 正在等待 15 美元的捐赠目标。

第一个被自然想到的解决方案可能是使用互斥锁。updater goroutine 会每秒增加余额。另一方面，listener goroutine 会循环，直到达到他们的捐赠目标：

```go
type Donation struct {
    mu      sync.RWMutex
    balance int
}
donation := &Donation{}

// Listener goroutines
f := func(goal int) {
    donation.mu.RLock()
    for donation.balance < goal {
        donation.mu.RUnlock()
        donation.mu.RLock()
    }
    fmt.Printf("$%d goal reached\n", donation.balance)
    donation.mu.RUnlock()
}
go f(10)
go f(15)

// Updater goroutine
go func() {
    for {
        time.Sleep(time.Second)
        donation.mu.Lock()
        donation.balance++
        donation.mu.Unlock()
    }
}()
```

我们使用互斥锁保护对共享变量 `donation.balance` 的访问。如果我们运行此示例，它将按预期工作：

```shell
$10 goal reached
$15 goal reached
```

主要问题以及使这种实现变得糟糕的原因是繁忙的循环。实际上，每个listener goroutine 都会一直循环，直到达到他们的捐赠目标，这将浪费大量 CPU 周期并使 CPU 使用量巨大。我们需要找到更好的解决方案。

让我们退后一步，看看我们需要什么。每当余额更新时，我们必须找到一种从 updater goroutine 发出信号的方法。如果我们考虑 Go 中的信号命令，我们应该考虑通道。因此，让我们尝试使用通道原语的另一个版本：

```go
type Donation struct {
    balance int
    ch      chan int
}

donation := &Donation{ch: make(chan int)}

// Listener goroutines
f := func(goal int) {
    for balance := range donation.ch {
        if balance >= goal {
            fmt.Printf("$%d goal reached\n", balance)
            return
        }
    }
}
go f(10)
go f(15)

// Updater goroutine
for {
    time.Sleep(time.Second)
    donation.balance++
    donation.ch <- donation.balance
}
```

每个 listener goroutine 接收到一个共享通道。同时，每当余额更新时，updater goroutine 都会发送消息。但是，如果我们尝试此解决方案，可能会出现以下输出：

```shell
$11 goal reached
$15 goal reached
```

当余额为 10 美元而不是 11 美元时，应该通知第一个 goroutine？所以发生了什么事？

原因是发送到通道的消息仅被一个 goroutine 接收。在我们的示例中，如果第一个 goroutine 在第二个之前从通道接收，则可能发生以下情况：

![](https://img.exciting.net.cn/68.png)

多个 goroutine 从共享通道接收的默认分发模式是轮询。如果一个 goroutine 还没有准备好接收消息（不是在通道上处于等待状态），它可以改变；在这种情况下，Go 会将消息分发到下一个可用的 goroutine。

在任何情况下，每个消息都由一个 goroutine 接收。因此，在这个例子中，第一个 goroutine 没有收到 $10 消息，但第二个收到了。只有通道关闭是可以广播到多个 goroutine 的事件。然而，这里我们不想关闭通道，因为更新程序 goroutine 不能再发送消息了。

此外，在这种情况下使用通道还有另一个问题。只要达到捐赠目标，listener goroutine 就会返回。因此，updater goroutine 必须知道所有侦听器何时停止接收到通道的消息。否则，它最终会变满并阻止发件人。一个可能的解决方案是在混合中添加一个 `sync.WaitGroup` 以及，但这会使解决方案更加复杂。

理想情况下，我们需要找到一种方法来在余额更新到多个 goroutine 时重复广播通知。幸运的是，Go 有一个解决方案：`sync.Cond`。让我们首先讨论理论，然后我们将看到如何使用这个原语解决我们的问题。

参考官方文档：

> Cond 实现了一个条件变量，一个 goroutine 的集合点，等待或宣布一个事件。
> 
> ‑‑ 同步包文档

条件变量是等待某个条件的线程（这里是 goroutine）的容器。在我们的示例中，条件是余额更新。每当余额更新时，updater goroutine 将广播通知，而 listener goroutine 将等待更新。此外，`sync.Cond` 依赖于 `sync.Locker` （一个`*sync.Mutex` 或 `*sync.RWMutex`）来防止数据竞争。这将是一个可能的实现：

```go
type Donation struct {
    cond    *sync.Cond
    balance int
}

donation := &Donation{
    cond: sync.NewCond(&sync.Mutex{}),
}

// Listener goroutines
f := func(goal int) {
    donation.cond.L.Lock()
    for donation.balance < goal {
        donation.cond.Wait()
    }
    fmt.Printf("%d$ goal reached\n",
    donation.balance)
    donation.cond.L.Unlock()
}
go f(10)
go f(15)

// Updater goroutine
for {
    time.Sleep(time.Second)
    donation.cond.L.Lock()
    donation.balance++
    donation.cond.L.Unlock()
    donation.cond.Broadcast()
}
```

首先，我们使用 `sync.NewCond` 创建一个 `*sync.Cond` 并提供一个 `*sync.Mutex`。那么，listener goroutine 和 updater goroutine 呢？

listener goroutines 循环，直到达到捐赠余额。在循环中，我们使用 `Wait` 方法，该方法将阻塞直到满足条件。

> **Note** 让我们确保在此处理解术语条件。在这个在上下文中，我们不是在谈论捐赠目标条件，而是在更新余额。因此，它是两个 listener goroutine 共享的单个条件变量。

对 `Wait` 的调用必须在可能听起来很奇怪的关键部分内完成。锁不会阻止其他 goroutine 也等待相同的条件吗？实际上，`Wait` 的实现如下：

* 解锁互斥锁 
* 暂停 goroutine 并等待通知 
* 通知到达时锁定互斥锁

因此，在listener goroutine 中，我们将有两个关键部分：

* 在 `for donation.balance < goal` 条件下访问 `donation.balance` 变量的时候。
* 在 `fmt.Printf` 函数中访问 `donation.balance` 变量的时候。

这样，对共享的 `donation.balance` 变量的所有访问都受到保护。

现在，updater goroutine 怎么样？余额更新在关键部分内完成，以防止数据竞争。然后，我们调用 `Broadcast` 方法，它会在每次余额更新时唤醒所有等待该条件的 goroutine。

因此，如果我们运行这个示例，它将打印出我们期望的结果：

```shell
10$ goal reached
15$ goal reached
```

在我们的实现中，条件变量基于正在更新的余额。因此，每次进行新的捐赠时，侦听器变量都会唤醒，以检查他们的捐赠目标是否达到。此解决方案可防止我们出现在重复检查中会消耗 CPU 周期的繁忙循环。

我们还要注意使用 `sync.Cond` 时可能存在的一个缺点。例如，当我们向 `chan struct` 发送通知时，即使没有活动的接收者，消息也会被缓冲，从而保证最终会收到此通知。将 `sync.Cond` 与 `Broadcast` 方法一起使用会唤醒当前等待该条件的所有 goroutines，如果没有，通知将被错过。这也是我们必须牢记的一个基本原则。

> **Note** 在这里，这段代码不会阻塞，因为没有 goroutine 等待在这个通道中接收消息。这与 `Signal()` 的原理相同。
> 
> 我们还应该注意一种使用 `Signal()` 而不是 `Broadcast` 来只唤醒一个 goroutine 的方法。就语义而言，它与以非阻塞方式在 `chan struct` 中发送消息相同：

```go
ch := make(chan struct{})
    select {
    case ch <- struct{}{}:
    default:
}
```

Go 中的信号命令可以通过通道来实现。但是，我们应该知道，只有一个通道关闭是多个 goroutine 可以捕获的事件。然而，这只能发生一次。因此，如果我们反复向多个 goroutine 发送通知，`sync.Cond` 是一个解决方案。该原语基于条件变量，这些条件变量设置等待特定条件的线程容器。使用 `sync.Cond`，我们可以广播将唤醒所有等待条件的 goroutine 的信号。

现在让我们使用 `golang.org/x` 和 `errgroup` 包扩展我们对并发原语的了解。