## 12.7 Go 语言的内联优化

内联是指用这个函数的主体替换一个函数调用。如今，内联是由编译器自动完成的。了解内联的基础知识也可以成为优化应用程序特定代码路径的一种方式。

首先，让我们看一个内联的具体示例，使用一个简单的 `sum` 函数对两种 `int` 类型求和：

```go
func main() {
    a := 3
    b := 2
    s := sum(a, b)
    println(s)
}

func sum(a int, b int) int {
    return a + b
}
```

如果我们使用 `gcflags` 运行 `go build`，我们将访问编译器做出的关于 `sum` 函数的决定：

```shell
$ go build -gcflags "-m=2"
./main.go:10:6: can inline sum with cost 4 as:
    func(int, int) int { return a + b }
...
./main.go:6:10: inlining call to sum func(int, int) int { return a + b }
```

在这里，编译器决定内联对 `sum` 的调用。因此，之前的代码将被替换为：

```go
func main() {
    a := 3
    b := 2
    s := a + b
    println(s)
}
```

但是，内联仅适用于具有一定复杂性的函数，也称为内联预算。否则，编译器会通知我们该函数太复杂而无法内联：

```shell
./main.go:10:6: cannot inline foo: function too complex:
    cost 84 exceeds budget 80
```

内联有两个主要好处。首先，它消除了函数调用的开销（尽管自 Go 1.17 和基于寄存器的调用约定以来，开销有所减轻）。其次，它允许编译器进行进一步的优化。例如，在内联函数之后，编译器可以决定最初应该在堆上转义的变量可能会留在栈上。

现在的问题是：如果这是编译器自动应用的优化，那么作为 Go 开发人员，我们为什么还要关心它呢？答案在于栈中内联的概念。

中间栈内联是关于调用其他函数的内联函数。在 Go 1.9 之前，仅考虑内联叶函数。现在，由于中间栈内联，以下 `foo` 函数也可以内联：

```go
func main() {
    foo()
}

func foo() {
    x := 1
    bar(x)
}
```

由于 `foo` 函数不太复杂，编译器可以内联它的调用：

```go
func main() {
    x := 1
    bar(x)
}
```

感谢中栈内联，作为 Go 开发人员，我们现在可以使用快速路径内联的概念来优化应用程序，以区分快速路径和慢速路径。让我们看一个在 `sync.Mutex` 实现中发布的具体示例，以了解此概念的工作原理。

在中间栈内联之前，`Lock` 方法的实现如下：

```go
func (m *Mutex) Lock() {
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        // Mutex isn't locked
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }

    // Mutex is already locked
    var waitStartTime int64
    starving := false
    awoke := false
    iter := 0
    old := m.state
    for {
        // ...
    }
    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

我们可以在这里区分两条主要路径：

* 如果互斥量未锁定（`atomic.CompareAndSwapInt32` 为真），快速路径。
* 如果互斥体已经被锁定（`atomic.CompareAndSwapInt32` 为假），慢路径。

但是，无论采用何种路径，函数都不能内联，因为它很复杂。然而，为了利用中间栈内联，`Lock` 方法被重构，以便慢速路径存在于特定函数中：

```go
func (m *Mutex) Lock() {
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }
    m.lockSlow()
}

func (m *Mutex) lockSlow() {
    var waitStartTime int64
    starving := false
    awoke := false
    iter := 0
    old := m.state
    for {
        // ...
    }
    
    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

由于此更改，现在可以内联 `Lock` 方法。这种方法的好处是现在可以锁定一个尚未锁定的互斥体，而无需支付调用函数的开销（它带来了大约 5% 的速度提升）。关于慢路径，当互斥体已经被锁定时，它没有改变。以前需要一个函数调用来执行这个逻辑，现在它仍然是一个函数调用，这次是 `lockSlow`。

因此，这种优化技术是关于区分快速路径和慢速路径。如果可以内联快速路径但不能内联慢速路径，我们可以在专用函数中提取慢速路径。因此，如果不超过内联预算，我们的函数将成为内联的候选者。

内联不仅仅是我们不应该关心的隐形编译器优化。如本节所示，了解内联如何工作以及如何访问编译器的决定可以成为使用快速路径内联技术进行优化的途径。实际上，如果执行快速路径，在专用函数中提取慢速路径会阻止函数调用。

下一节将深入研究常见的诊断工具，以了解在我们的 Go 应用程序中应该优化什么。