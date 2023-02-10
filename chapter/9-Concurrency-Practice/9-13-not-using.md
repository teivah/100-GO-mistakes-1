## 9.13 多个 goroutine 并行执行，使用 errgroup 聚合返回错误

不管是哪种编程语言，重新发明轮子都不是一个好主意。同样，找到重新实现如何启动多个 goroutine 并聚合错误的代码库也很常见。然而，Go 生态系统中的一个包旨在支持这种常见的用例。让我们深入研究它并理解为什么它应该成为 Go 开发人员工具集的一部分。

`golang.org/x` 是一个提供标准库扩展的存储库。在不同的子存储库中，有一个 `sync` 库包含一个方便的包：`errgroup`。

假设我们必须处理以下函数。我们接收一些我们想用来调用外部服务的数据作为参数。然而，由于某些限制，我们不能每次都使用不同的子集进行一次调用，而是多次调用。此外，这些调用将并行进行：

![](https://img.exciting.net.cn/69.png)

如果在调用过程中出现一个错误，我们希望将其返回。如果出现多个错误，我们只想返回其中一个。让我们尝试仅使用标准并发原语编写实现的框架：

```go
func handler(ctx context.Context, circles []Circle) ([]Result, error) {
    results := make([]Result, len(circles))
    wg := sync.WaitGroup{}
    wg.Add(len(results))

    for i, circle := range circles {
        i := i
        circle := circle
        go func() {
            defer wg.Done()

            result, err := foo(ctx, circle)
            if err != nil {
                // ?
            }
            results[i] = result
        }()
    }

    wg.Wait()
    // ...
}
```

在这里，我们决定使用一个 `sync.WaitGroup` 来等待所有的 goroutine 完成并在一个切片中处理聚合。这是一种方法；另一个可能是将每个部分结果发送到一个通道并将它们聚合到另一个 goroutine 中。如果需要排序，主要的挑战是重新排序传入的消息。因此，我们决定采用最简单的方法和共享切片。

> **Note** 当每个 goroutine 写入一个特定的索引时，这个实现是不会触发数据竞争的。

但是，有一个关键情况我们还没有解决。如果 `foo`（在新的 goroutine 中进行的调用）返回错误怎么办？我们应该如何处理？有不同的选项，例如：

* 就像 `results` 切片一样，我们可能会有一些错误在 goroutine 之间共享。如果出现错误，每个 goroutine 都会写入该切片。我们将不得不迭代这个切片在父 goroutine 中确定是否发生错误（O(n) 时间复杂度）。
* 另一种方法可能是让 goroutine 通过共享互斥锁访问单个错误变量。
* 我们还可以考虑共享一个错误通道，父 goroutine 将接收并处理这些错误。

无论选择何种选项，它都会使解决方案变得相当复杂。出于这个原因，设计和开发了 `errgroup` 包。

它导出单个 `WithContext` 函数，该函数在给定上下文的情况下返回 `*Group` 结构。该结构为一组 goroutine 提供同步、错误传递和上下文取消。它只导出两种方法：

* `Go` 在新的 goroutine 中触发调用。
* `Wait` 阻塞，直到所有 goroutine 都完成。如果有的话，它会返回第一个非零错误。

让我们使用 `errgroup` 重写解决方案。首先，我们需要导入 `errgroup` 包：

```shell
$ go get golang.org/x/sync/errgroup
```

这是实现：

```go
func handler(ctx context.Context, circles []Circle) ([]Result, error) {
    results := make([]Result, len(circles))
    g, ctx := errgroup.WithContext(ctx)

    for i, circle := range circles {
        i := i
        circle := circle
        g.Go(func() error {
            result, err := foo(ctx, circle)
            if err != nil {
                return err
            }
            results[i] = result
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

首先，我们通过提供父上下文创建一个 `*errgroup.Group`。在每次迭代中，我们使用 `g.Go` 在新的 goroutine 中触发调用。这个方法接受一个 `func() error` 作为输入，一个封装了对 `foo` 的调用并处理结果和错误的闭包。作为主要区别，如果我们得到一个错误，我们会从这个闭包中返回它。然后，`g.Wait` 允许我们等待所有的 goroutine 完成。

这个解决方案本质上比第一个更直接（这只是部分的，因为我们还没有处理错误）。我们不必依赖额外的并发原语，而且 `errgroup.Group` 足以解决我们的用例。

我们尚未解决的另一个好处是共享上下文。假设我们必须触发三个并行调用：

* 第一次调用在一毫秒内返回错误
* 第二次和第三次调用返回结果，或五秒后返回一个错误

在我们的例子中，我们想要返回一个错误，如果有的话。因此，等待第二次和第三次调用完成是没有意义的。使用 `errgroup.WithContext` 创建用于所有并行调用的共享上下文。由于第一次调用在一毫秒内返回错误，它将取消上下文，从而取消其他 goroutine。因此，我们不必等待五秒钟来返回错误。这是使用 `errgroup` 时的另一个好处。

> **Note** `g.Go` 调用的过程必须是 *上下文感知* 的；否则，取消上下文不会有任何效果。

总之，当我们必须触发多个 goroutine 并处理错误和上下文传播时，`errgroup` 是否可以成为我们的解决方案可能值得考虑。正如我们所见，这个包启用了一组 goroutines 的同步，并提供了处理错误和共享上下文的答案。

本章的最后一节将讨论 Go 开发人员在复制同步类型时常犯的错误。