## 6.6 忽略如何评估defer参数和接收器

我们在上一节中提到，`defer` 语句会延迟调用的执行，直到周围的函数返回。Go 开发人员常犯的一个错误是不了解如何评估参数。我们将分两个小节深入研究这个问题：一个与函数或方法参数有关，第二个与方法接收器有关。

### 6.6.1 参数评估

为了说明如何使用 `defer` 评估参数，让我们来看一个具体的例子。一个函数必须调用两个函数 `foo` 和 `bar`。同时，它必须处理有关执行的状态：

* `StatusSuccess` 如果 `foo` 和 `bar` 都没有返回错误。
* `StatusErrorFoo` 如果 `foo` 返回错误。
* `StatusErrorBar` 如果 `bar` 返回错误。

我们将使用此状态进行多项操作。例如，通知另一个 `goroutine` 并增加计数器。为了避免在每个 `return` 语句之前重复这些调用，我们将使用 `defer`。 这是我们的第一个实现：

```go
const (
        StatusSuccess  = "success"
        StatusErrorFoo = "error_foo"
        StatusErrorBar = "error_bar"
)

func f() error {
        var status string
        defer notify(status)
        defer incrementCounter(status)

        if err := foo(); err != nil {
                status = StatusErrorFoo
                return err
        }

        if err := bar(); err != nil {
                status = StatusErrorBar
                return err
        }

        status = StatusSuccess
        return nil
}
```

首先，我们声明一个 `status` 变量。然后，我们使用 `defer` 推迟对 `notify` 和 `incrementCounter` 的调用。在整个函数中，根据执行路径，我们相应地更新`status`。

但是，如果我们尝试一下这个函数，我们会注意到，无论执行路径如何，`notify` 和`incrementCounter` 总是以相同的状态被调用：一个空字符串。这怎么可能？

对于 defer 函数中的参数求值，有一点需要理解：参数是立即求值的，而不是在周围的函数返回后。在我们的示例中，我们将 `notify(status)` 和 `incrementStatus(status)` 称为 defer 函数。 因此，在我们使用 defer 的阶段，一旦 `f` 返回 `status` 的当前值，Go 将延迟这些调用执行；因此，一个空字符串。

如果我们想继续使用 `defer`，我们该如何解决这个问题？有两种领先的解决方案。

第一个解决方案是将字符串指针传递给 defer 函数：

```go
func f() error {
            var status string
            defer notify(&status)
            defer incrementCounter(&status)
    
            // The rest of the function unchanged
            if err := foo(); err != nil {
                    status = StatusErrorFoo
                    return err
            }
            if err := bar(); err != nil {
                status = StatusErrorBar
                return err
            }
            
            status = StatusSuccess
            return nil
}
```

我们会根据情况不断更新 `status`，但现在，`notify` 和 `incrementCounter` 会收到一个字符串指针。为什么它有效？

使用 `defer` 立即评估参数； 这里是 `status` 的地址。是的，`status` 本身在整个函数中都会被修改，但它的地址保持不变，无论分配如何。因此，如果 `notify` 或 `incrementCounter` 使用字符串指针引用的值，它将按预期工作。但是，此解决方案需要更改两个函数的签名，这可能并不总是可行。

还有另一种可能的解决方案：调用作为 defer 语句的闭包。提醒一下，闭包是一个匿名函数值，它从其主体外部引用变量。传递给 defer 函数的参数会立即被评估。然而，我们必须知道 defer 闭包引用的变量是在闭包执行期间评估的（因此，当周围的函数返回时）。

这是一个说明延迟闭包如何工作的示例。闭包将引用两个变量，一个作为函数参数，第二个作为其主体之外的变量：

```go
func main() {
        i := 0
        j := 0
        defer func(i int) {
                fmt.Println(i, j)
        }(i)
        i++
        j++
}
```

在这里，闭包使用 `i` 和 `j` 变量。`i` 作为函数参数传递，因此会立即对其进行评估。相反，`j` 引用闭包体外部的变量，因此在执行闭包时对其进行评估。如果我们运行这个例子，它将打印 `0 1`。

因此，我们可以使用闭包来实现我们函数的新版本：

```go
func f() error {
    var status string
    defer func() {
        notify(status)
        incrementCounter(status)
    }()
    
    // The rest of the function unchanged
}
```

在这里，我们将对 `notify` 和 `incrementCounter` 的调用封装在一个闭包中。 此闭包从其主体外部引用状态变量。因此，`status` 将在闭包执行后进行评估，而不是在我们调用 `defer` 时进行评估。

此解决方案也将起作用，并且不需要 `notify` 和 `incrementCounter` 更改其签名。

现在，在带有指针或值接收器的方法上使用 `defer` 怎么样？让我们深入研究这些问题。

### 6.6.2 指针和值接收器

在 *不知道要使用哪种类型的接收器* 中，我们讨论了接收器可以是值也可以是指针。 当我们在方法上使用 `defer` 时，与参数评估相关的相同逻辑也适用：接收者也立即被评估。 让我们了解两种接收器类型的影响：

首先，这是一个示例，它使用 defer 调用值接收器上的方法，但之后改变此接收器：

```go
func main() {
        s := Struct{id: "foo"}
        defer s.print()
        s.id = "bar"
}

type Struct struct {
        id string
}

func (s Struct) print() {
        fmt.Println(s.id)
}
```

我们将调用推迟到 `print` 方法。 与参数一样，调用 `defer` 会使接收者立即被评估。 因此， `defer` 将延迟方法的执行，它包含一个 `id` 字段等于 `foo` 的结构。 因此，此示例打印 `foo` 。

相反，如果指针是接收器，则调用 `defer` 后接收器的潜在变化将是可见的：

```go
func main() {
        s := &Struct{id: "foo"}
        defer s.print()
        s.id = "bar"
}

type Struct struct {
        id string
}

func (s *Struct) print() {
        fmt.Println(s.id)
}
```

`s` 接收器也立即被评估。但是，它会导致复制指针。因此，对指针引用的结构所做的更改是可见的。此示例打印 `bar`。

总之，我们必须提醒，当在函数或方法上调用 `defer` 时，调用的参数会立即被计算。如果我们想改变提供给 `defer` 的参数，我们可以使用指针或闭包。对于一个方法，接收者也立即被评估；因此，行为取决于接收者是值还是指针。