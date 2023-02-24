## 11.3 测试的 parallel 与 shuffle 模式

在运行测试时，go 命令可以接受一组标志来影响测试的执行方式。一个常见的错误是没有意识到这些标志并错过了可以带来更快执行或更好地发现可能错误的方法的机会。让我们深入研究其中的两个标志：`parallel` 和 `shuffle`。

### 11.3.1 并行(parallel)

我们应该注意的第一种执行模式是并行。它允许并行运行特定的测试，这非常有用，例如，加速长时间运行的测试。

我们可以通过调用 `t.Parallel` 来标记测试必须并行运行：

```go
func TestFoo(t *testing.T) {
    t.Parallel()
    // ...
}
```

使用 `t.Parallel` 标记测试时，它将与所有其他并行测试一起并行执行。不过，在执行方面，Go 首先一个一个地运行所有顺序测试。然后，一旦顺序测试完成，它就会执行并行测试。

例如，以下代码包含三个测试，但其中只有两个被标记为并行运行：

```go
func TestA(t *testing.T) {
    t.Parallel()
    // ...
}

func TestB(t *testing.T) {
    t.Parallel()
    // ...
}

func TestC(t *testing.T) {
    // ...
}
```

运行此文件的测试将为我们提供以下日志：

```shell
=== RUN   TestA
=== PAUSE TestA
=== RUN   TestB
=== PAUSE TestB
=== RUN   TestC
--- PASS: TestC (0.00s)
=== CONT  TestA
--- PASS: TestA (0.00s)
=== CONT  TestB
--- PASS: TestB (0.00s)
PASS
```

`TestC` 是第一个被执行的。首先记录 `TestA` 和 `TestB`，但它们被暂停，等待 `TestC` 完成。然后，两者都被恢复，然后并行执行。

默认情况下，可以同时运行的最大测试数等于 `GOMAXPROCS` 值。例如，如果我们想要序列化测试，或者在长时间运行的测试执行大量 I/O 的情况下增加这个数字，我们可以使用 `‑parallel` 标志更改这个值：

```shell
$ go test -parallel 16 .
```

这里，最大并行测试数设置为 16。

现在让我们看看运行 Go 测试时的另一种模式：shuffle。

### 11.3.2 shuffle（随机）

从 Go 1.17 开始，现在可以随机化测试和基准测试的执行顺序。

这样做的理由是什么？编写测试时的最佳实践是将它们隔离。例如，它们不应该依赖于执行顺序或共享变量。这些隐藏的依赖关系可能意味着一个可能的测试错误，或者更糟糕的是，一个在测试中不会被捕获的错误。为了防止这种情况，我们应该使用 `‑shuffle` 标志来随机化测试。我们可以提供 `on` 或 `off` 来启用或禁用测试随机播放（默认禁用）：

```shell
$ go test -shuffle=on -v .
```

然而，在某些情况下，我们希望以相同的顺序再次运行测试。例如，如果在 CI 期间测试失败，我们可能希望在本地重现错误。为此，我们可以传递用于随机化测试的种子，而不是将 `on` 传递给 `shuffle` 标志。

通过启用详细模式（`-v`），我们可以在运行随机测试时访问此种子值：

```shell
$ go test -shuffle=on -v .
-test.shuffle 1636399552801504000
=== RUN   TestBar
--- PASS: TestBar (0.00s)
=== RUN   TestFoo
--- PASS: TestFoo (0.00s)
PASS
ok      teivah  0.129s
```

在这里，我们随机执行了测试，但是 `go test` 打印了种子值：1636399552801504000。为了强制以相同的顺序运行测试，我们可以将这个种子值提供给 `shuffle` ：

```shell
$ go test -shuffle=1636399552801504000 -v . -test.shuffle 1636399552801504000
=== RUN   TestBar
--- PASS: TestBar (0.00s)
=== RUN   TestFoo
--- PASS: TestFoo (0.00s)
PASS
ok      teivah  0.129s
```

测试以相同的顺序执行：首先是 `TestBar`，然后是 `TestFoo`。

一般来说，我们应该对现有的测试标志保持谨慎，并随时了解最新 Go 版本中的新功能。并行运行测试是减少运行所有测试的总体执行时间的绝佳方式。此外，`shuffle` 模式可以帮助我们发现隐藏的依赖关系，这些依赖关系可能意味着在以相同顺序运行测试时出现测试错误甚至不可见的错误。