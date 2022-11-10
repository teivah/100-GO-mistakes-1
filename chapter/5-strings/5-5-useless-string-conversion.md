## 5.5 无用的字符串转换

在选择使用字符串或 `[]byte` 时，大多数程序员为了方便起见倾向于使用字符串。然而，大多数 I/O 实际上是用 `[]byte` 完成的。例如， `io.Reader`、`io.Writer` 和 `io.ReadAll` 与 `[]byte` 一起工作，而不是字符串。 因此，使用字符串意味着额外的转换，尽管 `bytes` 包包含许多与 `strings` 包相同的操作。

让我们看一个我们不应该做的例子。我们将实现一个 `getBytes` 函数，该函数将 `io.Reader` 作为输入，从中读取，然后调用 `sanitize` 函数。清理将通过修剪所有前导和尾随空格来完成。这是 `getBytes` 的骨架：

```go
func getBytes(reader io.Reader) ([]byte, error) {
        b, err := io.ReadAll(reader)
        if err != nil {
                return nil, err
        }
        // Call sanitize
}
```

我们调用 `ReadAll` 并将字节切片分配给 `b`。那么，我们如何实现 `sanitize` 功能呢？一种选择可能是使用 `strings` 包创建一个 `sanitize(string)` 字符串函数：

```go
func sanitize(s string) string {
        return strings.TrimSpace(s)
}
```

现在，回到 `getBytes`，当我们操作 `[]byte` 时，我们必须先将其转换为字符串，然后再调用 `sanitize`。然后，我们必须将结果转换回 `[]byte`，因为 `getBytes` 返回一个字节切片：

```go
return []byte(sanitize(string(b))), nil
```

这个实现有什么问题？我们必须付出额外的代价，将 `[]byte` 转换为字符串，然后将字符串转换为 `[]byte`。在内存方面，这些转换中的每一个都需要额外的分配。实际上，即使字符串由 `[]byte` 支持，将 `[]byte` 转换为字符串也需要字节切片的副本。这意味着新的内存分配和为所有字节创建副本。

> **Note** 我们可以使用以下代码测试从 `[]byte` 创建字符串会导致副本的事实：

```go
b := []byte{'a', 'b', 'c'}
s := string(b)
b[1] = 'x'
fmt.Println(s)
```

> 运行此代码不会打印 `axc` 而是打印 `abc`。事实上，在Go中，字符串是不可变的。

那么，我们应该如何实现 `sanitize` 功能呢？代替接受并返回一个字符串，我们应该操作字节切片：

```go
func sanitize(b []byte) []byte {
        return bytes.TrimSpace(b)
}
```

`bytes` 包还有一个 `TrimSpace` 函数来修剪所有前导和尾随空格。现在，在调用方，它不需要任何额外的转换：

```go
return sanitize(b), nil
```

正如我们所提到的，大多数 I/O 是使用 `[]byte` 完成的，而不是字符串。当我们想知道我们应该使用字符串还是 `[]byte` 时，让我们回想一下，使用 `[]byte` 并不一定不那么方便。事实上，`strings` 包的所有导出函数在 `bytes` 包中也有它们的替代方案：`Split`、`Count`、`Contains`、`Index` 等。因此，无论是在做 I/O 还是在其他情况下，我们都应该首先检查我们是否不能使用字节而不是字符串实现整个工作流程，并避免额外转换的代价。

本章的最后一节将讨论子字符串操作有时如何导致内存泄漏的情况。