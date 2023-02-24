## 11.8 编写不准确的基准

一般来说，我们永远不应该猜测性能。在编写一些优化时，很多因素可能会发挥作用，即使我们对结果有强烈的看法，测试它也不是一个坏主意。然而，编写基准测试并不简单。事实上，编写不准确的基准并因此做出错误的假设非常简单。本节的目标是深入研究导致不准确的常见和具体陷阱。

在讨论这些陷阱之前，让我们简要记住 Go 中的基准测试是如何工作的。

基准的框架如下：

```go
func BenchmarkFoo(b *testing.B) {
    for i := 0; i < b.N; i++ {
        foo()
    }
}
```

函数名称以 Benchmark 前缀开头。在 `for` 循环中调用被测函数 (`foo`)。`b.N` 表示可变的迭代次数。运行基准测试时，Go 将尝试使其与请求的基准测试时间相匹配。基准时间默认设置为一秒，可以使用 `-benchtime` 标志进行更改。`b.N` 从一开始；如果基准测试在一秒内完成，`b.N` 会增加，然后基准测试再次运行，直到 `b.N` 与 `benchtime` 大致匹配：

```shell
$ go test -bench=.
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkFoo-4 73 16511228 ns/op
```

在这里，基准测试大约需要一秒钟，而 `foo` 执行了 73 次，平均执行时间为 16511228 纳秒。我们可以使用 `‑benchtime` 更改基准时间：

```shell
$ go test -bench=. -benchtime=2s
BenchmarkFoo-4 150 15832169 ns/op
```

现在，`foo` 的执行量大约是之前基准测试的两倍。

现在让我们深入研究这些常见的陷阱。

### 11.8.1 不重置或暂停计时器

在某些情况下，我们需要在基准循环之前执行一些操作。有时，这些操作可能需要相当长的时间（例如，生成大量数据）并且可能对基准测试结果产生重大影响：

```go
func BenchmarkFoo(b *testing.B) {
    expensiveSetup()
    for i := 0; i < b.N; i++ {
        functionUnderTest()
    }
}
```

在这种情况下，我们可以做的是在进入循环之前使用 `ResetTimer` 方法：

```go
func BenchmarkFoo(b *testing.B) {
    expensiveSetup()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        functionUnderTest()
    }
}
```

调用 `ResetTimer` 将自测试开始以来经过的基准时间和内存分配计数器归零。这样，可以从测试结果中丢弃昂贵的设置。

现在，如果我们不仅要在每次循环迭代中执行一些昂贵的设置，该怎么办？

```go
func BenchmarkFoo(b *testing.B) {
    for i := 0; i < b.N; i++ {
        expensiveSetup()
        functionUnderTest()
    }
}
```

在这里，我们无法重置计时器，因为它会在每次循环迭代期间执行。然而，可以停止和恢复基准计时器以围绕对 `expensiveSetup` 的调用：

```go
func BenchmarkFoo(b *testing.B) {
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        expensiveSetup()
        b.StartTimer()
        functionUnderTest()
    }
}
```

在这里，我们暂停基准计时器以执行昂贵的设置，然后恢复它。

> **Note** 关于这种方法有一个问题需要记住：如果被测函数与设置函数相比执行速度太快，则基准测试可能需要很长时间才能完成。原因是它需要超过一个用户秒才能达到 `benchtime`。事实上，计算基准时间完全基于 `functionUnderTest` 的执行时间。因此，如果我们在每次循环迭代中等待很长时间，这将使基准测试比一秒慢。如果我们想保持基准，一种可能的缓解方法是减少 `benchtime`。

让我们确保使用计时器方法来保持基准测试的准确性。

### 11.8.2 对微基准做出错误假设

微基准是关于测量一个微小的计算单元，有时可以毫不费力地对它做出错误的假设。

例如，假设我们在使用 `atomic.StoreInt32` 还是 `atomic.StoreInt64` 之间犹豫不决（假设我们将处理的值始终适合 32 位）。我们想编写一个基准来比较这两个函数：

```go
func BenchmarkAtomicStoreInt32(b *testing.B) {
    var v int32
    for i := 0; i < b.N; i++ {
        atomic.StoreInt32(&v, 1)
    }
}

func BenchmarkAtomicStoreInt64(b *testing.B) {
    var v int64
    for i := 0; i < b.N; i++ {
        atomic.StoreInt64(&v, 1)
    }
}
```

如果我们运行这个例子，我们可以得到一个可能的输出：

```shell
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkAtomicStoreInt32
BenchmarkAtomicStoreInt32-4     197107742                5.682 ns/op
BenchmarkAtomicStoreInt64
BenchmarkAtomicStoreInt64-4     213917528                5.134 ns/op
```

我们可以很容易地认为这个基准是理所当然的，并决定使用 `atomic.StoreInt64`，因为它看起来更快。现在，为了做一个 *公平* 的基准测试来恢复顺序，所以首先测试 `atomic.StoreInt64`，然后是 `atomic.StoreInt32`。这可能是另一个可能的输出：

```shell
BenchmarkAtomicStoreInt64
BenchmarkAtomicStoreInt64-4 224900722 
5.434 ns/op
BenchmarkAtomicStoreInt32
BenchmarkAtomicStoreInt32-4 230253900 
5.159 ns/op
```

现在，`atomic.StoreInt32` 有更好的结果，那么发生了什么？

在微基准测试的情况下，许多因素会影响结果，例如运行基准测试的机器活动、电源管理、热缩放、指令序列的更好缓存对齐。我们必须记住，许多因素，即使在我们的 Go 项目范围之外也会影响结果。

> **Note** 我们应该确保执行基准测试的机器是空闲的。但是，外部进程可能会在后台运行，这可能会影响基准测试结果。出于这个原因，存在一些工具（例如 `perflock`）来限制基准测试可以消耗多少 CPU。例如，我们可以使用总可用 CPU 的 70% 运行基准测试。 这样，它为操作系统和其他进程提供了 30% 的空间。因此，这将有助于降低机器活动因素对结果的影响。

第一个选项可以是使用 `‑benchtime` 选项增加基准测试时间。与概率论中的大数定律类似，多次运行基准测试应该会趋于接近其预期值（假设我们忽略了指令缓存和此类机制的好处）。

另一种选择是在经典基准测试工具之上使用外部工具。 其中之一是 `golang.org/x` 存储库的一部分：`benchstat`。 该工具允许我们计算和比较有关基准执行的统计数据。 

例如，让我们使用 `-count` 选项运行基准测试十次，并将输出通过管道传输到特定文件：

```shell
$ go test -bench=. -count=10 | tee stats.txt
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkAtomicStoreInt32-4     234935682                5.124 ns/op
BenchmarkAtomicStoreInt32-4     235307204                5.112 ns/op
// ...
BenchmarkAtomicStoreInt64-4     235548591                5.107 ns/op
BenchmarkAtomicStoreInt64-4     235210292                5.090 ns/op
// ...
```

然后我们可以在这个文件上运行 `benchstat`：

```shell
$ benchstat stats.txt
name                time/op
AtomicStoreInt32-4  5.10ns ± 1%
AtomicStoreInt64-4  5.10ns ± 1%
```

现在，我们观察到结果是相同的：两个函数平均需要 5.10 ns 才能完成。它还显示了给定基准执行之间的百分比变化：± 1%。这个指标告诉我们两个基准都非常稳定，让我们对计算出的平均结果更有信心。因此，我们可以得出结论，对于我们测试的用法（在特定机器上的特定 Go 版本中），它具有与 `atomic.StoreInt64` 相似的执行时间，而不是得出结论说 `atomic.StoreInt32` 更快或更慢。

一般来说，我们应该对微观基准保持谨慎。事实上，许多因素会对结果产生重大影响，并导致潜在的错误假设。增加基准测试时间或使用 `benchstat` 等工具重复执行基准测试和计算统计数据可能是限制外部因素并获得更准确结果从而得出更好结论的有效方法。

另外，让我们强调一下，如果最终运行应用程序的系统不同，我们应该小心在给定机器上执行的微基准测试的结果。事实上，生产系统的行为可能与我们运行微基准测试的完全不同。

### 11.8.3 不注意编译器优化

另一个与编写基准有关的常见错误是被一些编译器优化所愚弄，因为它也可能导致错误的基准假设。

在本节中，我们将以 Go 问题 14813 为例，该问题也由 Dave Cheney 强调，其中包含一个人口计数函数（一个计算位数设置为 1 的函数）：

```go
const m1 = 0x5555555555555555
const m2 = 0x3333333333333333
const m4 = 0x0f0f0f0f0f0f0f0f
const h01 = 0x0101010101010101

func popcnt(x uint64) uint64 {
    x -= (x >> 1) & m1
    x = (x & m2) + ((x >> 2) & m2)
    x = (x + (x >> 4)) & m4
    return (x * h01) >> 56
}
```

此函数接受并返回一个 `uint64`。为了对这个函数进行基准测试，可以编写以下内容：

```go
func BenchmarkPopcnt1(b *testing.B) {
    for i := 0; i < b.N; i++ {
        popcnt(uint64(i))
    }
}
```

但是，如果我们执行这个基准测试，我们会得到一个非常低的结果：

```shell
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkPopcnt1-4      1000000000               0.2858 ns/op
```

0.28 纳秒的持续时间大约是一个时钟周期，因此这个数字低得不合理。这里的问题是开发人员对编译器优化不够谨慎。实际上，在这种情况下，被测函数非常简单，可以作为内联的候选函数。内联是一种优化，它用被调用函数的主体替换函数调用，它允许防止占用空间小的函数调用。内联函数后，编译器可以注意到调用没有副作用，并将其替换为以下基准：

```go
func BenchmarkPopcnt1(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // Empty
    }
}
```

基准现在是空的。因此，这就是我们得到接近一个时钟周期的结果的原因。为防止这种情况发生，最佳实践是遵循以下模式：

* 在每次循环迭代期间，将结果分配给局部变量（基准函 数上下文中的局部变量）
* 将最新结果分配给全局变量

在我们的例子中，这意味着编写以下基准：

```go
var global uint64

func BenchmarkPopcnt2(b *testing.B) {
    var v uint64
    for i := 0; i < b.N; i++ {
        v = popcnt(uint64(i))
    }
    global = v
}
```

`global` 是一个全局变量，而 `v` 是一个局部变量，其范围是基准函数。在每次循环迭代期间，我们将 `popcnt` 的结果分配给局部变量。然后，我们将最新的结果分配给全局变量。

> **Note** 我们可能会问，为什么不将 `popcnt` 调用的结果直接分配给 `global` 以简化测试？写入全局变量比写入局部变量慢（我们将在不理解堆栈与堆中深入研究这些概念）。因此，我们应该将每个结果写入一个局部变量，以限制每次循环迭代期间的足迹。

如果我们运行这两个基准测试，我们现在将获得显着的结果差异：

```shell
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkPopcnt1-4      1000000000               0.2858 ns/op
BenchmarkPopcnt2-4      606402058                1.993 ns/op
BenchmarkPopcnt2-4      606402058                1.993 ns/op 
```

`BenchmarkPopcnt2` 是基准的准确版本。它保证我们避免了内联优化，它可以人为地降低执行时间甚至删除对被测函数的调用。因此，依赖 `BenchmarkPopcnt1` 的结果可能会导致错误的假设。

让我们记住避免编译器优化欺骗基准测试结果的模式：将被测函数的结 果分配给局部变量，然后将最新结果分配给全局变量。这样的最佳实践还可 以防止做出错误的假设。

### 11.8.4 被观察者效应愚弄

在物理学中，观察者效应是观察行为对被观察系统的扰动。这种影响也可以 在导致对结果做出错误假设的基准中看到。让我们看一个具体的例子，然后尝试减轻它。

我们想要实现一个接收 `int64` 元素矩阵的函数。这个矩阵有固定数量的 512 列，我们要计算前八列的总和。

![](https://img.exciting.net.cn/74.png)

为了优化，我们还想看看改变列数是否有影响，所以我们还要实现第二个 513 列的函数。实现如下：

```go
func calculateSum512(s [][512]int64) int64 {
    var sum int64
    for i := 0; i < len(s); i++ {
            for j := 0; j < 8; j++ {
                    sum += s[i][j]
            }
    }
    return sum
}

func calculateSum513(s [][513]int64) int64 {
    // Same implementation as calculateSum512
}
```

我们遍历每一行，然后遍历前八列，然后递增我们返回的 `sum` 变量。 `calculateSum513` 中的实现保持不变。

我们想对这些函数进行基准测试，以确定在给定固定行数的情况下哪个函数性能最高：

```go
const rows = 1000

var res int64

func BenchmarkCalculateSum512(b *testing.B) {
    var sum int64
    s := createMatrix512(rows)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sum = calculateSum512(s)
    }
    res = sum
}

func BenchmarkCalculateSum513(b *testing.B) {
    var sum int64
    s := createMatrix513(rows)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sum = calculateSum513(s)
    }
    res = sum
}
```

我们只想创建一次矩阵以限制结果的足迹。因此， `createMatrix512` 和 `createMatrix513` 在循环外被调用。我们可能期望结果与我们只想迭代前八列的结果相似；但是，情况并非如此（在我的机器上）：

```shell
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkCalculateSum512-4        81854             15073 ns/op
BenchmarkCalculateSum513-4       161479              7358 ns/op
```

具有 513 列的第二个基准测试大约快 50%。同样，由于我们只迭代前八列，这个结果可能非常令人惊讶。

要了解这种差异，我们需要了解有关 CPU 缓存的基础知识。简而言之，CPU由不同的缓存（通常是L1、L2和L3）组成。这些高速缓存用于降低从主内存访问数据的平均成本。在某些情况下，CPU 可以从主存中取出数据并将其复制到 L1。在这种情况下，CPU 会尝试将矩阵的子集 `calculateSum` 感兴趣（每行的前八列）提取到 L1 中。然而，在一种情况下（513 列）它适合内存，而在另一种情况下（512 列），它不适合。

> **Note** 这不是本章解释原因的范围。但是，我们将在 *不了解 CPU 缓存* 中深入研究这个问题。

回到基准测试，主要问题是我们在两种情况下都重复使用相同的矩阵。由于该函数重复了数千次，因此我们不会测量接收一个简单的新矩阵的函数执行情况。相反，我们测量一个函数，该函数获取一个矩阵，该矩阵已经在缓存中存在单元的子集。因此，由于`calculateSum513` 导致更少的缓存未命中，它具有更好的执行时间。

这是观察者效应的一个例子。因为我们一直在观察一个重复调用的 CPU‑bound 函数，CPU 缓存可能会发挥作用并显着影响结果。

在这个例子中，为了防止这种影响，我们不应该重用矩阵，而是在每次测试期间创建一个：

```go
func BenchmarkCalculateSum512(b *testing.B) {
    var sum int64
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        s := createMatrix512(rows)
        b.StartTimer()
        sum = calculateSum512(s)
    }
    res = sum
}
```

现在在每次循环迭代期间创建一个新矩阵。如果我们再次运行此基准测 试（并调整 `benchtime`，否则执行时间太长），结果现在彼此更接近：

```shell
cpu: Intel(R) Core(TM) i5-7360U CPU @ 2.30GHz
BenchmarkCalculateSum512-4         1116             33547 ns/op
BenchmarkCalculateSum513-4          998             35507 ns/op
```

因此，我们现在没有错误地假设 `calculateSum513` 更快，而是看到两者在接收新矩阵时会导致相似的结果。

正如我们在本节中看到的，当我们重用相同的矩阵时，CPU 缓存会显着影响结果。为了防止它，我们必须在每次循环迭代期间创建一个新矩阵。一般来说，我们应该记住，观察被测函数在某些情况下可能会导致结果的显着差异，尤其是在低级优化很重要的 CPU 绑定函数的微基准测试环境中。在每次迭代期间强制基准重新创建数据可能是防止这种影响的好方法。

在本章的最后一部分，让我们深入探讨有关 Go 测试的常见技巧。