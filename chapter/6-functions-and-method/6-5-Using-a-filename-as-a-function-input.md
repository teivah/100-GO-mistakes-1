## 6.5 使用文件名作为函数输入

在创建需要读取文件的新函数时，传递文件名不被认为是最佳实践，并且可能会产生负面影响，例如使单元测试更难编写。让我们深入研究这个问题并了解如何克服它。

我们想实现一个函数来计算文件中的空行数。实现此功能的一种可能方法是接受文件名并使用 `bufio.NewScanner` 扫描和检查每一行：

```go
func countEmptyLinesInFile(filename string) (int, error) {
        file, err := os.Open(filename)
        if err != nil {
                return 0, err
        }
        // Handle file closure

        scanner := bufio.NewScanner(file)
        for scanner.Scan() {
                // ...
        }
}
```

我们从文件名打开一个文件。然后，我们使用 `bufio.NewScanner` 扫描每一行（默认情况下，它会每行拆分输入）。

这个函数会做我们期望它做的事情。事实上，只要提供的文件名有效，我们就会从中读取并返回空行数。所以有什么问题？

假设我们要实现单元测试以涵盖以下情况：

* 名义上的案例
* 一个空文件
* 仅包含空行的文件

每个单元测试都需要在我们的 Go 项目中创建一个文件。函数越复杂，我们可能想要添加的案例越多，我们将创建的文件就越多。在某些情况下，我们甚至可能不得不创建数十个文件，这很快就会变得难以管理。

此外，人们还可能会指出此功能不可重用。例如，如果我们必须实现相同的逻辑，但要计算 HTTP 请求的空行数，我们将不得不复制主逻辑：

```go
func countEmptyLinesInHTTPRequest(request http.Request) (int, error) {
        scanner := bufio.NewScanner(request.Body)
        // Copy the same logic
}
```

克服这些限制的一种可能方法是使函数接受 `*bufio.Scanner`（由 `bufio.NewScanner` 返回的输出）。 事实上，从我们创建 `scanner` 变量的那一刻起，这两个函数就具有相同的逻辑。它会起作用的。然而，在 Go 中，惯用的方式是从读者的抽象开始。

让我们编写一个新版本的 `countEmptyLines` 函数，它会接收一个 `io.Reader` 抽象：

```go
func countEmptyLines(reader io.Reader) (int, error) {
        scanner := bufio.NewScanner(reader)
        for scanner.Scan() {
                // ...
        }
}
```

由于 `bufio.NewScanner` 接受 `io.Reader`，我们可以直接传递 `reader` 变量。

那么有什么好处呢？ 首先，这个函数抽象了数据源。是文件吗？HTTP 请求？socket 输入？对函数来说并不重要。由于 `*os.File` 和 `http.Request` 的 `Body` 字段实现了 `io.Reader`，我们可以重用相同的函数，而不管输入类型如何。

另一个好处与测试有关。我们提到每个测试用例创建一个文件很快就会变得很麻烦。现在 `countEmptyLines` 接受 `io.Reader`，我们可以通过从字符串创建 `io.Reader` 来实现单元测试：

```go
func TestCountEmptyLines(t *testing.T) {
    emptyLines, err := countEmptyLines(strings.NewReader(
    `foo
                                bar
        
                                baz
                                `))
    // Test logic
}
```

在这个测试中，我们直接从字符串文字中使用 `strings.NewReader` 创建了一个 `io.Reader`。因此，我们不必为每个测试用例创建一个文件。每个测试用例可以是独立的，提高了测试的可读性和可维护性，因为读者不必打开另一个文件来查看内容。

在大多数情况下，接受文件名作为从文件读取的函数输入应被视为代码异味（在特定函数中，例如 `os.Open` 除外）。正如我们所见，它使单元测试更加复杂，因为我们可能必须创建多个文件。此外，它降低了函数的可重用性（尽管并非所有函数最终都将被重用）。使用 `io.Reader` 接口抽象数据源。无论输入是文件、字符串、HTTP 还是 gRPC 请求，都可以重用并轻松测试实现。

对于本章的最后一部分，让我们讨论一个与 `defer` 相关的常见错误：如何评估函数/方法参数和方法接收器。