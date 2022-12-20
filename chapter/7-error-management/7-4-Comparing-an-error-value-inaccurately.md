## 7.4 不准确地比较错误值

本节类似于我们刚刚讨论的那一节，但带有标记错误（错误值）。 首先，我们将定义标记错误传达的信息。 然后，我们将看到如何将错误与值进行比较。

标记错误是定义为全局变量的错误：

```go
import "errors"
 
var ErrFoo = errors.New("foo")
```

通常，约定是以 `Err` 开头，后跟错误类型；这里 `ErrFoo`,标记错误传达了 *预期* 的错误。但是我们所说的预期错误是什么意思呢？让我们在 SQL 库的上下文中讨论它。

我们想设计一个 `Query` 方法，允许对数据库执行查询。此方法返回行的一部分。找不到行的情况下应该怎么处理呢？这里我们有两个选择：

* 返回标记值；例如，一个 nil 切片（想想 `strings.Index`，如果子字符串不存在则返回标记值 -1）
* 或者返回客户端可以检查的特定错误

让我们采用第二种方法。因此，如果未找到任何行，我们的方法可以返回特定错误。我们可以将此错误归类为预期错误，因为允许传递不返回任何行的请求。相反，如果出现网络问题或连接轮询错误等错误，我们将面临意外错误。这并不意味着我们不想处理意外错误；这意味着在语义上，它们传达了不同的含义。

如果我们看一下标准库，我们会发现很多标记错误的例子：

* `sql.ErrNoRows`:当查询没有返回任何行时返回（这正是我们的情况）
* `io.EOF`:当没有更多输入时由 `io.Reader` 返回可用的。

这就是标记错误背后的一般原则。它们传达了客户期望检查的预期错误。因此，作为一般准则：

* 预期错误应设计为错误值（标记错误）：`var ErrFoo = errors.New("foo")`
* 意外错误应设计为错误类型：`type BarError struct{ ... }` 用 `BarError` 实现 `error` 接口。

让我们回到常见的错误。我们如何将错误与特定值进行比较？我们可以使用 `==` 运算符：

```go
err := query()
if err != nil {
        if err == sql.ErrNoRows {
                // ...
        } else {
                // ...
        }
}
```

在这里，我们调用了一个 `query` 函数并得到了一个错误。检查错误是否为 `sql.ErrNoRows` 是使用 `==` 运算符完成的。

但是，与我们在上一节中讨论的方式相同，也可以包装标记错误。 如果使用 `fmt.Errorf` 和 `%w` 指令包装 `sql.ErrNoRows`，则 `err == sql.ErrNoRows` 将始终为 false。

同样，从 1.13 版开始的 Go 提供了答案。我们已经看到了 `errors.As` 是如何被用来检查一个类型的错误的。对于错误值，我们应该使用它的对应方法：`errors.Is`。 让我们重写前面的例子：

```go
err := query()
if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
                // ...
        } else {
                // ...
        }
}
```

即使错误是使用 `%w` 包装的，也要使用 `errors.Is` 来替代 `==` 运算符去做比较工作。

总之，如果我们在应用程序中使用 `%w` 指令和 `fmt.Errorf` 进行错误包装，则不应该使用 `==` 而是使用 `errors.Is` 来检查特定值的错误。因此，即使标记错误被包装，`errors.Is` 也能够递归地解包并将链中的每个错误与提供的值进行比较。

现在，是时候讨论错误处理最重要的方面之一了：不要两次处理错误。