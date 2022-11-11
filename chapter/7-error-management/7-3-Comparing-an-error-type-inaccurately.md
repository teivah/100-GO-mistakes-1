## 7.3 不准确的错误类型比较

上一节介绍了一种使用 `%w` 指令包装错误的可能方法。然而，一旦我们开始使用它，改变我们检查特定错误类型的方式也很重要。否则，我们可能会不准确地处理错误。

让我们讨论一个具体的例子。我们将编写一个 HTTP 处理程序以从 ID 返回交易金额。我们的处理程序将解析请求以获取 ID 并从数据库中检索金额。我们的实现可能在两种情况下失败：

* 如果 ID 无效（字符串长度不是五个字符）
* 如果程序数据库失败

在前一种情况下，我们想要返回 StatusBadRequest (400)，而在后一种情况下，我们想要 ServiceUnavailable (503)。为此，我们将创建一个 `transientError` 类型来标记错误是暂时的。父处理程序将检查错误类型。如果错误是 `transientError `错误，它将返回 503；否则，400。

让我们首先关注错误类型定义和处理程序将调用的函数：

```go
type transientError struct {
        err error
}

func (t transientError) Error() string {
        return fmt.Sprintf("transient error: %v", t.err)
}

func getTransactionAmount(transactionID string) (float32, error) {
        if len(transactionID) != 5 {
                return 0, fmt.Errorf("id is invalid: %s", transactionID)
        }

        amount, err := getTransactionAmountFromDB(transactionID)
        if err != nil {
                return 0, transientError{err: err}
        }
        return amount, nil
}
```

如果标识符无效，`getTransactionAmount` 使用 `fmt.Errorf` 返回错误。但是，如果从数据库中获取交易金额失败，它会将错误包装到 `transientError` 错误类型中。

现在，让我们编写将检查错误类型以返回适当的 HTTP 状态代码的 HTTP 处理程序：

```go
func handler(w http.ResponseWriter, r *http.Request) {
        transactionID := r.URL.Query().Get("transaction")

        amount, err := getTransactionAmount(transactionID)
        if err != nil {
                switch err := err.(type) {
                case transientError:
                        http.Error(w, err.Error(), http.StatusServiceUnavailable)
                default:
                        http.Error(w, err.Error(), http.StatusBadRequest)
                }
                return
        }

        // Write response
}
```

使用错误类型的 `switch`，我们返回适当的 HTTP 状态代码：400 表示错误请求或 503 表示暂时错误。

此代码完全有效。但是，现在让我们假设我们想要对 `getTransactionAmount` 执行一个小的重构。`TransactionError` 不会由 `getTransactionAmount` 返回，而是由 `getTransactionAmountFromDB`返回。`getTransactionAmount` 现在将使用 `%w` 指令包装此错误：

```go
func getTransactionAmount(transactionID string) (float32, error) {
        // Check transaction ID validity

        amount, err := getTransactionAmountFromDB(transactionID)
        if err != nil {
                return 0, fmt.Errorf("failed to get transaction %s: %w",
                        transactionID, err)
        }
        return amount, nil
}

func getTransactionAmountFromDB(transactionID string) (float32, error) {
        // ...
        if err != nil {
                return 0, transientError{err: err}
        }
        // ...
}
```

如果我们运行这段代码，我们会注意到，无论错误情况如何，我们总是会返回 400。这意味着永远不会遇到这种情况下的 `case Transient` 错误。我们如何解释这种行为？

重构之前，`transientError` 被 `getTransactionAmount` 返回。

![](https://img.exciting.net.cn/45.png)

重构之后，`transientError` 由 `getTransactionAmountFromDB` 返回：

![](https://img.exciting.net.cn/46.png)

`getTransactionAmount` 直接返回的不是一个 `transientError`：它是一个包装了 `transientError` 的错误。因此，`case transientError` 现在是假(false)的。

出于这个确切目的，Go 1.13 还附带了一个包装错误的指令，以及一种使用 `errors.As` 检查包装错误是否属于某种类型的方法。如果链中的错误与预期类型匹配，则此函数递归地解包错误并返回 true。

现在让我们使用 `errors.As` 重写调用者的实现：

```go
func handler(w http.ResponseWriter, r *http.Request) {
        // Get transaction ID
    
        amount, err := getTransactionAmount(transactionID)
        if err != nil {
                if errors.As(err, &transientError{}) {
                        http.Error(w, err.Error(), http.StatusServiceUnavailable)
                } else {
                        http.Error(w, err.Error(), http.StatusBadRequest)
                }
                return
        }

        // Write response
}
```

在这个新版本中，我们去掉了 switch case 类型，现在使用 `errors.As`。这个函数要求第二个参数（目标错误）是一个指针。否则，该函数将编译但在运行时出现恐慌。使用 `errors.As`，无论运行时错误是直接是 `transientError` 类型还是错误包装的 `transientError` ，它都能匹配然后返回 503。

综上所述，如果我们依赖 Go 1.13 的错误包装，我们必须使用 `errors.As` 来检查错误是否为特定类型。这样，无论错误是由我们调用的函数直接返回还是包装在错误中，`error.As` 都将能够递归地解开我们的主要错误并查看其中一个错误是否是特定类型。

我们刚刚看到了如何比较错误类型；现在是时候比较一个错误值。