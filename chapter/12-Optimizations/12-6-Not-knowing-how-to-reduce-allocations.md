## 12.6 减少内存分配对程序的优化

减少分配是加速 Go 应用程序的常用优化技术。在本书中，我们已经介绍了一些减少堆分配数量的方法：

* 未优化的字符串连接部分（Under-optimized strings concatenation）：使用 `strings.Builder` 代替 `+` 运算符连接字符串。
* 无用字符串转换（Useless string conversion）：尽可能避免将 `[]byte` 转换为字符串。
* 低效的切片和 map 初始化（ Inefficient slice initialization 和 Inefficient map initialization）：如果长度已知，则预分配切片和 map。
* 更好的数据结构对齐以减少结构大小（不知道数据对齐）。

作为本节的一部分，我们将深入探讨三种常见的方法：

* 变更我们的 API
* 依赖编译器优化
* 使用诸如 `sync.Pool` 等工具。

### 12.6.1 API 变更

第一种选择是仔细处理我们提供的 API。让我们以 `io.Reader` 接口为例：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

`Read` 方法接受一个切片并返回读取的字节数。现在，想象一下如果 `io.Reader` 接口的设计方式相反：传递一个表示必须读取多少字节的 `int` 并返回一个切片：

```go
type Reader interface {
    Read(n int) (p []byte, err error)
}
```

从语义上讲，它不会有任何问题。但是，在这种情况下，返回的切片会自动逃逸到堆中。实际上，我们将处于上一节中描述的共享案例中。

Go 设计者采用共享向下方法来防止自动将切片转义到堆中。因此，由调用者提供切片。这并不一定意味着这个切片不会被转义。事实上，编译器可能已经决定这个切片不能留在堆栈上。但是，这取决于调用者来处理它，而不是调用 `Read` 方法带来的约束。

因此，有时即使是 API 的微小变化也会对分配产生积极影响。在设计 API 时，让我们始终了解上一节中描述的转义分析规则，并在需要时使用 `gcflags` 来理解编译器的决定。

### 12.6.2 编译器优化

Go 编译器的目标之一是尽可能优化我们的代码。这是一个关于 map 的具体例子。

在 Go 中，无法使用切片作为键类型来定义 map。在某些情况下，尤其是在执行 I/O 的应用程序中，我们可以接收我们想用作键的 `[]byte` 数据。因此，我们必须先将其转换为字符串，并且可以编写以下代码：

```go
type cache struct {
    m map[string]int
}

func (c *cache) get(bytes []byte) (v int, contains bool) {
    key := string(bytes)
    v, contains = c.m[key]
    return
}
```

由于 `get` 函数接收到 `[]byte` 切片，我们将其转换为 `key` 字符串以查询 map。

但是，我们应该注意到 Go 编译器在我们使用 `string(bytes)` 查询 map 的情况下实现了特定的优化：

```go
func (c *cache) get(bytes []byte) (v int, contains bool) {
    v, contains = c.m[string(bytes)]
    return
}
```

尽管代码几乎相同（我们直接调用 `string(bytes)` 而不是传递变量），但在这种情况下，编译器将避免将此字节转换为字符串。因此，第二个版本将比第一个更快。

这个例子说明了一个看起来相似的函数的两个版本可能会在 Go 编译器的工作之后产生不同的汇编代码。我们还应该注意可能的编译器优化以优化应用程序。同时，我们应该关注未来的 Go 版本，以检查语言中是否添加了新的优化。

### 12.6.3 sync.Pool

如果我们想解决分配数量的问题，另一个改进的途径是考虑使用 `sync.Pool`。

首先，我们应该了解 `sync.Pool` 不是缓存：我们没有可以设置的固定大小或最大容量。相反，它是一个重用公共对象的池。

让我们想象一下以下情况。我们要实现一个接收 `io.Writer` 的 `write` 函数，调用一个函数来获取 `[]byte` 切片，然后将其写入 `io.Writer`。我们的代码将如下所示（为清楚起见，我们将省略错误处理）：

```go
func write(w io.Writer) {
    b := getResponse()
    _, _ = w.Write(b)
}
```

在这里，`getResponse` 在每次调用时返回一个新的 `[]byte` 切片。如果我们想通过潜在地重用这个切片来减少分配的数量怎么办？我们将假设所有响应的最大大小为 1024 字节。在这种情况下，我们可以使用 `sync.Pool`。

创建一个 `sync.Pool` 需要一个 `func() any` 工厂函数；例如：

![](https://img.exciting.net.cn/108.png)

`sync.Pool` 公开了两个方法：

* `Get() any`: 从池中获取对象
* `Put(any)`:将对象返回到池中

如果池为空，则使用 `Get` 将创建一个新对象，否则将重用一个对象。然后，在使用该对象后，我们可以使用 `Put` 将其放回池中。这是一个示例，其中包含先前定义的工厂，当池为空时使用 `Get`，当池不为空时使用 `Put` 和 `Get`：

![](https://img.exciting.net.cn/109.png)

所以一个问题，什么时候会从池中排出对象？它没有特定的方法：它依赖于 GC。事实上，在每次 GC 之后，池中的对象都会被销毁。

如果我们回到我们的示例，假设我们可以更新 `getResponse` 函数以将数据写入给定切片而不是创建切片，我们现在可以实现依赖于池的 `write` 方法的另一个版本：

```go
var pool = sync.Pool{
    New: func() any {
        return make([]byte, 1024)
    },
}

func write(w io.Writer) {
    buffer := pool.Get().([]byte)
    buffer = buffer[:0]
    defer pool.Put(buffer)

    getResponse(buffer)
    _, _ = w.Write(buffer)
}
```

首先，我们使用 `sync.Pool` 结构定义一个新池，并设置工厂函数创建一个长度为 1024 个 元素的新 `[]byte`。

在 `write` 函数中，我们尝试从池中检索一个缓冲区。如果池是空的，它将创建一个新的；否则，它将从池中选择一个任意缓冲区并将其返回。一个关键步骤是使用 `buffer[:0]` 重置缓冲区，因为该切片可能已被使用。然后，我们推迟调用 `Put` 以将切片放回池中。

在这个新版本中，调用 `write` 不会导致为每个调用创建一个新的 `[]byte` 切片。相反，我们可以重用现有分配的切片。在最坏的情况下，例如在一次 GC 之后，它会创建一个新的缓冲区；但是，摊销分配成本降低了。

综上所述，如果我们经常分配许多相同类型的对象，我们可以考虑使用 `sync.Pool`。实际上，`sync.Pool` 是一组临时对象，可以帮助我们防止重复重新分配相同类型的数据。此外，`sync.Pool` 可以安全地同时被多个 goroutine 使用。

现在让我们讨论内联的概念，以了解它不一定是我们不应该关心的编译器优化。