## 9.8 忘记字符串格式化引入的坑

格式化字符串是开发人员的常见操作，无论是返回错误还是记录消息。但是，在并发应用程序中工作时，很容易忘记字符串格式化的潜在副作用。本节将看到两个具体示例：一个取自 etcd 存储库导致数据竞争，另一个导致死锁情况。

### 9.8.1 etcd 数据竞争

etcd 是一个用 Go 实现的分布式键值存储。它在包括 Kubernetes 在内的许多项目中用于存储所有集群数据。它提供了与集群交互的 API。例如，`Watcher` 接口用于在数据更改时得到通知：

```go
type Watcher interface {
    // Watch watches on a key or prefix. The watched events will be returned
    // through the returned channel.
    // ...
    Watch(ctx context.Context, key string, opts ...OpOption) WatchChan
    Close() error
}
```

API 依赖于 gRPC 流。如果您不熟悉它，它是一种在客户端和服务器之间不断交换数据的技术。服务器必须维护使用此功能的所有客户端的列表。因此，`Watcher` 接口由包含所有活动流的 `watcher` 结构实现：

```go
type watcher struct {
    // ...

    // streams hold all the active gRPC streams keyed by ctx value.
    streams map[string]*watchGrpcStream
}
```

映射的键基于调用 `Watch` 方法时提供的上下文：

```go
func (w *watcher) Watch(ctx context.Context, key string,
    opts ...OpOption) WatchChan {
    // ...
    ctxKey := fmt.Sprintf("%v", ctx)
    // ...
    wgs := w.streams[ctxKey]
    // ...
```

`ctxKey` 是映射的键，从客户端提供的上下文格式化。从使用值（`context.WithValue`）创建的上下文格式化字符串时，Go 将尝试访问以读取所有值。在这种情况下，etcd 开发人员发现提供给 `Watch` 的上下文在某些情况下是包含可变值（例如，指向结构的指针）的上下文。他们发现一个 goroutine 正在更新其中一个上下文值，而另一个正在执行 `Watch` 的情况；因此在读取上下文值时访问。这导致了数据竞赛。

修复是不依赖 `fmt.Sprintf` 来格式化映射的键，以防止遍历和读取上下文中的包装值链。相反，解决方案是实现一个自定义的 `streamKeyFromCtx` 函数来从特定的上下文值中提取键，该值是不可变的。

> **Note** 上下文中潜在的可变值可能会引入额外的复杂性以防止数据竞争。这可能是一个需要谨慎考虑的设计决策。

这个例子说明我们必须小心并发应用程序中字符串格式化的副作用，在这种情况下是数据竞争。在下面的示例中，我们将看到导致死锁的情况。

### 9.8.2 死锁

假设我们必须处理一个可以同时访问的 `Customer` 结构。我们将使用 `sync.RWMutex` 来保护访问，无论是读取还是写入。我们将实现一个 `UpdateAge` 方法来更新客户的年龄并检查年龄是否为正。同时，我们将实现 `Stringer` 接口。

你能看到这个代码中的问题是什么，一个 `Customer` 结构暴露了一个 `UpdateAge` 方法并实现了 `fmt.Stringer` 接口：

```go
type Customer struct {
    mutex sync.RWMutex
    id    string
    age   int
}

func (c *Customer) UpdateAge(age int) error {
    c.mutex.Lock()
    defer c.mutex.Unlock()

    if age < 0 {
        return fmt.Errorf("age should be positive for customer %v", c)
    }

    c.age = age
    return nil
}

func (c *Customer) String() string {
    c.mutex.RLock()
    defer c.mutex.RUnlock()
    return fmt.Sprintf("id %s, age %d", c.id, c.age)
}
```

这里的问题可能不是那么简单。如果提供的 `age` 为负数，我们将返回错误。当错误被格式化时，在接收器上使用 `%s` 指令，它将调用 `String` 方法来格式化 `Customer`。然而，由于 `UpdateAge` 已经获取了互斥锁，`String` 方法将无法获取它：

![](https://img.exciting.net.cn/66.png)

因此，这种情况导致死锁情况。如果所有 goroutine 也都处于休眠状态，则会导致 panic:

```shell
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_SemacquireMutex(0xc00009818c, 0x10b7d00, 0x0)
...
```

那么我们应该如何处理这种情况呢？首先，它说明了单元测试的重要性。在这种情况下，有人可能会争辩说，创建一个负年龄的测试是不值得的，因为逻辑非常简单。但是，如果没有适当的测试覆盖率，我们可能会错过这个问题。

这里可以改进的一件事是限制互斥锁的范围。实际上，在 `UpdateAge` 中，我们首先获取锁并检查输入是否有效。我们应该做相反的事情：首先检查输入，如果输入有效，我们应该获得锁。这具有减少潜在副作用的好处，但也可能对性能产生影响：仅在需要时获取锁，而不是之前：

```go
func (c *Customer) UpdateAge(age int) error {
    if age < 0 {
        return fmt.Errorf("age should be positive for customer %v", c)
    }

    c.mutex.Lock()
    defer c.mutex.Unlock()

    c.age = age
    return nil
}
```

在我们的例子中，只有在检查了年龄后才锁定互斥锁可以避免死锁情况。实际上，如果年龄为负数，则调用 `String` 时无需事先锁定互斥锁。

但是，在某些情况下，限制互斥锁的范围并不总是那么简单，也不可能。在这些情况下，我们必须非常小心字符串格式。也许，我们想调用另一个不尝试获取互斥锁的函数，或者我们只想改变格式化错误的方式，使其不调用 `String` 方法。例如，以下代码不会导致死锁，因为我们只在直接访问 `id` 字段时记录客户 ID：

```go
func (c *Customer) UpdateAge(age int) error {
    c.mutex.Lock()
    defer c.mutex.Unlock()

    if age < 0 {
        return fmt.Errorf("age should be positive for customer id %s", c.id)
    }

    c.age = age
    return nil
}
```

我们已经看到了两个具体的例子，一个是从上下文格式化一个键，另一个是返回一个将格式化结构的错误。在这两种情况下，格式化字符串都会导致问题，前一个示例中的数据竞争以及后一个示例中的死锁情况。因此，在并发应用程序中，我们应该对字符串格式化可能产生的副作用保持谨慎。

以下部分将深入研究同时调用时 `append` 的行为。
