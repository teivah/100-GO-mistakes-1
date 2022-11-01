## 4.6 在循环中使用 `defer`

`defer` 语句延迟调用的执行，直到周围的函数返回。它主要用于减少样板代码。例如，如果一个资源最终必须关闭，我们可以使用 `defer` 来避免在每次 `return` 之前重复关闭调用。然而，一个常见的错误是没有意识到在循环中使用 `defer` 的后果。让我们来看看这个问题。

我们将实现一个打开一组文件的函数，其中文件路径将通过通道接收。因此，我们将不得不遍历这个通道，打开文件并处理关闭。这是我们的第一个版本：

```go
func readFiles(ch <-chan string) error {
        for path := range ch {
                file, err := os.Open(path)
                if err != nil {
                        return err
                }

                defer file.Close()

                // Do something with file
        }
        return nil
}
```

> **Note** 我们将在 _Not handling defer_ 中讨论如何处理 `defer` 错误。

这种实现存在一个重大问题。事实上，我们必须记得 `defer` 会在 **周围** 函数返回时安排函数调用。在这种情况下，`defer` 调用不会在每次循环迭代期间执行，而是在 `readFiles` 函数返回时执行。如果 `readFiles` 没有返回，文件描述符将永远保持打开状态，导致泄漏。

有哪些选项可以解决此问题？

一种选择是摆脱延迟并手动处理文件关闭。然而，这意味着我们将不得不放弃 Go 工具集的一个方便的特性，因为我们处于一个循环中。 那么如果我们想继续使用 `defer` 有哪些选择呢？ 我们必须在每次迭代期间调用 `defer` 周围的另一个函数。

例如，我们可以实现一个 `readFile` 来保存每个接收到的新文件路径的逻辑：

```go
func readFiles(ch <-chan string) error {
        for path := range ch {
                if err := readFile(path); err != nil {
                        return err
                }
        }
        return nil
}

func readFile(path string) error {
        file, err := os.Open(path)
        if err != nil {
                return err
        }
        
        defer file.Close()
        
        // Do something with file
        return nil
}
```

在这个实现中，当 `readFile` 返回时调用 `defer` 函数，这意味着在每次迭代结束时。因此，在父 `readFiles` 函 数返回之前，我们不会保持打开文件描述符。

另一种方法可能是使 `readFile` 函数成为闭包：

```go
func readFiles(ch <-chan string) error {
        for path := range ch {
                err := func() error {
                        // ...
                        defer file.Close()
                        // ...
                }()
                if err != nil {
                        return err
                }
        }
        return nil
}
```

然而，本质上，它仍然是相同的解决方案：在每次迭代期间添加另一个外围函数来执行 `defer` 调用。普通的旧函数的优点是可能更清晰一些，我们也可以为它编写一个特定的单元测试。

使用 `defer` 时，我们必须记住，它会在周围函数返回时安排函数调用。 因此，在循环中调用 `defer` 将堆叠所有调用：它们不会在每次迭代期间执行，例如，如果循环没有终止，这可能会导致内存泄漏。 解决这个问题最方便的方法是在每次迭代期间引入另一个要调用的函数。 然而，如果性能至关重要，那么一个缺点是函数调用增加的开销。 如果我们在这种情况下并且想要防止这样的开销，我们应该摆脱 `defer` 并在循环之前手动处理 `defer` 调用。

