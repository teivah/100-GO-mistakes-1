## 9.14 拷贝同步类型变量引入的坑

`sync` 包提供了基本的同步原语，例如互斥体、条件变量和等待组。对于所有这些类型，有一个硬性规则要遵循：它们不应该被复制。让我们了解它的基本原理和可能存在的问题。

我们将创建一个线程安全的数据结构来存储计数器。它将包含表示每个计数器当前值的一个 `map[string]int` 变量。此外，我们将使用 `sync.Mutex`，因为访问必须受到保护。我们还添加一个增量方法来增加给定的计数器名称：

```go
type Counter struct {
    mu       sync.Mutex
    counters map[string]int
}
    
func NewCounter() Counter {
    return Counter{counters: map[string]int{}}
}

func (c Counter) Increment(name string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.counters[name]++
}
```

增量逻辑在关键部分内完成：在 `c.mu.Lock()` 和 `c.mu.Unlock()` 之间。让我们通过使用 `-race` 选项运行以下示例来尝试我们的方法，该示例启动两个 goroutine 并递增它们各自的计数器：

```go
counter := NewCounter()

go func() {
    counter.Increment("foo")
}()
go func() {
    counter.Increment("bar")
}()
```

如果我们运行这个示例，它将引发数据竞争：

```shell
==================
WARNING: DATA RACE
...
```

我们的 `Counter` 实现中的问题是因为互斥锁被复制了。事实上，因为 `Increment` 的接收者是一个值，所以每当我们调用 `Increment` 时，它都会执行 `Counter` 结构的副本，这也会复制互斥锁。

`sync` 类型不应该被赋值，这个规则同样使用以下类型：

* `sync.Cond`
* `sync.Map`
* `sync.Mutex`
* `sync.RWMutex`
* `sync.Once`
* `sync.Pool`
* `sync.WaitGroup`

因此，互斥锁不应该被复制。那么有哪些选择呢？

第一个是修改 `Increment` 方法的接收器类型：

```go
func (c *Counter) Increment(name string) {
    // Same code
}
```

更改接收器类型可避免在调用 `Increment` 时复制 `Counter`。因此，不会复制内部互斥锁。

如果我们想保留一个值接收器，第二个选择是将 `Counter` 中 `mu` 字段的类型更改为指针：

```go
type Counter struct {
    mu       *sync.Mutex
    counters map[string]int
}
    
func NewCounter() Counter {
    return Counter{
        mu: &sync.Mutex{},
        counters: map[string]int{},
    }
}
```

如果 `Increment` 有一个值接收者，它仍然会复制 `Counter` 结构。但是，由于 `mu` 现在是一个指针，它只会执行指针复制，而不是 `sync.Mutex` 的实际复制。因此，此解决方案还将防止数据争用。

> **Note** 我们还改变了 `mu` 的初始化方式。事实上，由于 `mu` 是一个指针，如果我们在创建 `Counter` 时省略它，它将被初始化为指针的零值：nil。因此，当调用 `c.mu.Lock()` 时，会导致 goroutine panic。

在以下某些情况下，我们可能会遇到无意复制 `sync` 字段的问题：

* 使用值接收器调用方法（如我们所见）
* 使用 `sync` 参数调用函数
* 使用包含 `sync` 字段的参数调用函数

在每种情况下，我们都应该保持非常谨慎。另外，让我们注意一些 linter 可以捕捉到这个问题。例如，使用 `go vet`：

```shell
$ go vet .
    ./main.go:19:9: Increment passes lock by value: Counter contains sync.Mutex
```

根据经验，当多个 goroutine 必须访问一个公共 `sync` 元素时，我们必须确保它们都依赖于同一个实例。此规则适用于 `sync` 包中定义的所有类型。使用指针是解决此问题的一种方法：通过使用指向 `sync` 元素的指针或指向包含 `sync` 元素的结构的指针。