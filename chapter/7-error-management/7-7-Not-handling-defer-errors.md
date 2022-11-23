## 7.7 不被处理的 defer 错误

不处理 defer 语句中的错误是 Go 开发人员经常犯的错误。让我们了解问题是什么以及可能的解决方案是什么。

在下面的示例中，我们将实现一个函数来查询数据库以获取给定客户 ID 的余额。我们将使 用 `database/sql` 和 `Query` 方法。

> **Note** 让我们不要过多地研究这个包是如何工作的；我们将在 *SQL 常见错误的标准库* 章节中进行处理。

这是一个可能的实现（我们将关注查询本身，而不是结果的解析）：

```go
const query = "..."
func getBalance(db *sql.DB, clientID string) (
    float32, error) {
    rows, err := db.Query(query, clientID)
    if err != nil {
        return 0, err
    }
    defer rows.Close()

    // Use rows
}
```

`rows` 是一个 `*sql.Rows` 类型。它实现了 `Closer` 接口：

```go
type Closer interface {
    Close() error
}
```

此接口包含一个返回错误的 `Close` 方法（我们还将在 *不关闭瞬态资源* 中深入研究此主题）。我们在上一节中提到应该始终处理错误。然而，在这种情况下，由 defer 调用返回的错误被忽略：

```go
defer rows.Close()
```

如上一节所述，如果我们不想处理错误，我们应该使用空白标识符显式忽略它：

```go
defer func() { _ = rows.Close() }()
```

这个版本更冗长，但从可维护性的角度来看更好，因为我们明确标记了我们忽略错误。

然而，在这种情况下，与其盲目地忽略 defer 调用中的所有错误，不如问问自己这是否是最好的方法。在这种情况下，调用 `Close()` 将在无法从池中释放数据库连接时返回错误。因此，忽略这个错误可能不是我们想要做的。最有可能的是，更好的选择是记录一条消息：

```go
defer func() {
        err := rows.Close()
        if err != nil {
                log.Printf("failed to close rows: %v", err)
        }
}()
```

在这种情况下，如果关闭 `rows` 失败，它将记录一条消息，以便我们知道它。

现在，如果我们不处理错误，而是将其传播给 `getBalance` 的错误，以便他决定如何处理它，该怎么办？这种方法怎么样？

```go
defer func() {
        err := rows.Close()
        if err != nil {
                return err
        }
}()
```

此实现无法编译。实际上，`return` 语句与匿名 `func()` 函数相关联，而不是与 `getBalance` 相关联。

如果我们想将 `getBalance` 返回的错误与 defer 调用中捕获的错误联系起来，我们必须使用命名结果参数。让我们重写第一个版本：

```go
func getBalance(db *sql.DB, clientID string) (
        balance float32, err error) {
        rows, err := db.Query(query, clientID)
        if err != nil {
                return 0, err
        }
        defer func() {
                err = rows.Close()
        }()
    
        if rows.Next() {
                err := rows.Scan(&balance)
                if err != nil {
                        return 0, err
                }
                return balance, nil
        }
        // ...
}
```

一旦正确创建了 `rows` 变量，我们将在匿名函数中推迟对 `rows.Close()` 的调用。此函数将错误分配给 `err` 变量，使用命名结果参数进行初始化。

这段代码可能看起来不错；但是，它有一个问题。事实上，如果 `rows.Scan` 返回错误，`rows.Close` 无论如何都会被执行 。然而，由于此调用覆盖了 `getBalance` 返回的错误，而不是返回错误，如果 `rows.Close` 成功返回，我们可能会返回 nil 错误 。换句话说，如果调用 `db.Query` 成功（函数的第一行），`getBalance` 返回的错误将始终是 `rows.Close` 返回的错误，这不是我们想要的。

实现的逻辑并不简单。如果 `rows.Scan` 成功：

* 如果 `rows.Close` 成功：不返回错误。
* 如果 `rows.Close` 失败：返回此错误。

但是，如果 `rows.Scan` 失败，逻辑会稍微复杂一些，因为我们可能需要处理两个错误：

* 如果 `rows.Close` 成功：从 `rows.Scan` 返回错误。
* 如果 `rows.Close` 失败：？

在 `rows.Scan` 和 `rows.Close` 都失败了，我们该怎么办？这里有不同的选择。例如，返回一个传达两个错误的自定义错误。我们将实施的另一个可能是返回 `rows.Scan` 错误但记录 `rows.Close` 错误。这是匿名函数的最终实现：

```go
defer func() {
        closeErr := rows.Close()
        if err != nil {
                if closeErr != nil {
                        log.Printf("failed to close rows: %v", err)
                }
                return
        }
        err = closeErr
}()
```

`rows.Close` 错误分配给另一个变量：`closeErr`。在将其分配给 `err` 之前，我们检查 `err` 是否不同于 nil。如果是这种情况，则 `getBalance` 已返回错误。在这种情况下，我们决定记录它并返回现有错误。

如前所述，应该始终处理错误。对于 defer 调用返回的错误，我们至少应该明确地忽略它。如果这还不够，我们可以决定通过记录错误或将错误传播给调用者来直接处理错误，如本节所示。