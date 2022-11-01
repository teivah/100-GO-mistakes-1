## 4.5 忽略 break 语句的工作方式

`break` 语句通常用于终止循环的执行。当循环与 `switch` 或 `select` 一起使用时，经常会看到开发人员犯了中断错误语句的错误。

让我们看一下下面的例子。 我们将实现一个 `switch` 在 `for` 循环内。 如果循环索引的值为 2，我们要中断循环：

```go
for i := 0; i < 5; i++ {
        fmt.Printf("%d ", i)

        switch i {
        default:
        case 2:
                break
        }
}
```

这段代码乍一看可能是正确的； 但是，它并没有达到我们的预期。实际上，`break` 语句不会终止 `for` 循环，而是 `switch` 语句。 因此，此代码从 0 迭代到 4，而不是从 0 到 2：`0 1 2 3 4`。

要记住的一个基本规则：`break` 语句终止最里面的 `for` 、 `switch` 或 `select` 语句的执行。 在前面的例子中：`switch` 语句。

那么我们如何编写这样一个打破循环而不是 `switch` 语句的代码呢？最惯用的方法是使用标签：

```go
loop:
for i := 0; i < 5; i++ {
        fmt.Printf("%d ", i)

        switch i {
        default:
        case 2:
                break loop
        }
}
```

在这里，我们将 `loop` 标签与 `for` 循环相关联。 然后，当我们向 `break` 语句提供循环标签时，它会中断循环，而不是 `switch`。因此，这个新版本将打印 `0 1 2` ，正如我们所期望的那样。

> **Note** 人们可能会质疑与标签的中断是否是惯用的，并将其视为一种花哨的 goto 语句。但是，情况并非如此，标准库中使用了这样的代码。例如，在 `net/http` 包中从缓冲区读取行时：

```go
readlines:
for {
        line, err := rw.Body.ReadString('\n')
        switch {
        case err == io.EOF:
                break readlines
        case err != nil:
                t.Fatalf("unexpected error reading from CGI: %v", err)
        }
        // ...
}
```
> 此示例使用带有 `readlines` 的富有表现力的标签来强调循环的目标。
> 因此，我们应该考虑打破使用标签作为 Go 中惯用方法的语句。

循环中的 `select` 也可能会中断错误的语句。在下面的代码中，我们希望在两种情况下使用 `select` 并在上下文取消时中断循环：

```go
for {
    select {
    case <-ch:
            // Do something
    case <-ctx.Done():
            break
    }
}
```

这里最里面的 `for` 、 `switch` 或 `select` 语句是 `select` 语句，而不是 `for` 循环。 因此，循环将重复。 同样，为了打破循环本身，我们可以使用标签来做到这一点：

```go
loop:
for {
        select {
        case <-ch:
                // Do something
        case <-ctx.Done():
                break loop
        }
}
```

现在，正如预期的那样，`break` 语句打破了循环，而不是 `select`。

> **Note** 我们还可以使用带有标签的 `continue` 来进入标签循环的下一次迭代。

在循环中使用 `switch` 或 `select` 语句时，我们应该保持谨慎。 使用 `break` 时，我们应该始终确保知道它将影响哪个语句。 正如我们所见，使用标签是强制破坏特定语句的惯用解决方案。

在本章的最后一节，我们将继续讨论循环，但这次与 `defer` 关键字结合使用。