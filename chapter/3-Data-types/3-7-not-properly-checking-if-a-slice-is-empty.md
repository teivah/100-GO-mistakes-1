## 3.7 没有正确检查切片是否为空

我们在上一节中已经看到，`nil`切片和空切片之间存在区别。考虑到这些概念，检查切片是否包含元素的惯用方法是什么？没有明确的答案可能会导致微妙的错误。

让我们考虑以下示例。我们将调用一个返回 `float32` 切片的 `getOperations` 函数。然后，我们只想在切片包含元素时调用 `handle` 函数。这是第一个（错误的）版本：

```go
func handleOperations(id string) {
        operations := getOperations(id)
        if operations != nil {
                handle(operations)
        }
}

func getOperations(id string) []float32 {
        operations := make([]float32, 0)

        if id == "" {
                return operations
        }

        // Add elements to operations
		return operations
}
```

在这个例子中，检查切片是否有元素是通过检查 `operations` 切片是否不为`nil`来完成的。然而，这段代码有问题。实际上，`getOperations` 永远不会返回 `nil` 切片。相反，它返回一个空切片。因此 `operations != nil` 检查将始终为真。

那么，在这种情况下我们该怎么办呢？第一种方法可能是说如果 `id` 为空，我们需要修改 `getOperations` 以返回一个 nil 切片：

```go
func getOperations(id string) []float32 {
        operations := make([]float32, 0)

        if id == "" {
                return nil
        }

        // Add elements to operations

        return operations
}
```

如果 `id` 为空，我们不返回 `operations`，而是返回`nil`。这样，我们实现的关于测试切片是否为`nil`的检查将匹配。

但是，这种方法并非在所有情况下都可以实行，因为有些时候我们并不允许修改函数的内部实现。例如，如果我们使用第三方库，我们不会创建只是为了将空切片更改为`nil`切片的`PR`。

那么我们如何检查一个切片是空的还是`nil`呢？解决方案是检查长度：

```go
func handleOperations(id string) {
    operations := getOperations(id)
    if len(operations) != 0 {
        handle(operations)
    }
}
```

我们在上一节中提到，根据定义，空切片的长度为零。同时`nil`切片始终为空。因此，通过检查切片的长度来检查，我们涵盖了所有场景：

* 如果切片为 nil： `len(operations) != 0` 将为`false`
* 如果切片不是 nil 而是空的： `len(operations) != 0` 也将是`false`

因此，检查长度是最好的选择，因为我们无法总是控制调用函数的内部实现。同时，正如`Go Wiki`所说，在设计接口时，我们应该避免区分`nil`和空切片，从而导致细微的编程错误。因此，在返回切片时，如果我们返回`nil`或空切片，我们既不应该产生语义上的差异，也不应该产生技术差异。对于调用者来说，两者都应该意味着同样的事情。

这个原理与 `map` 相同。要检查`map`是否为空，我们应该检查它的长度，而不是它是否为 nil。

在下一节中，我们将看到如何正确地制作切片副本。