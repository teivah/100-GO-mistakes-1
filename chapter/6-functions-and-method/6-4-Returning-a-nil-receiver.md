## 6.4 返回一个 nil 接收器引发的坑爹事件

我们将在本节中讨论返回接口的影响以及为什么它可能在某些情况下导致错误。这个错误可能是 Go 中最普遍的错误之一，因为它可能被认为是违反直觉的，至少在我们做到之前是这样。

让我们考虑以下示例。我们将处理 `Customer` 结构并实现一个 `Validate` 方法来执行完整性检查。然而，我们不想返回第一个错误，而是可能返回一个错误列表。为此，我们将创建一个自定义错误类型来传达多个错误：

```go
type MultiError struct {
        errs []string
}

func (m *MultiError) Add(err error) {
        m.errs = append(m.errs, err.Error())
}

func (m *MultiError) Error() string {
        return strings.Join(m.errs, ";")
}
```

`MultiError` 满足错误接口，因为它实现了 `Error()  string`。同时，它公开了一个 `Add` 方法来追加错误。使用此结构，我们可以通过以下方式实现 `Customer.Validate` 方法来检查客户的年龄和姓名。如果完整性检查没问题，我们想返回一个 nil 错误：

```go
func (c Customer) Validate() error {
        var m *MultiError

        if c.Age < 0 {
                m = &MultiError{}
                m.Add(errors.New("age is negative"))
        }
        if c.Name == "" {
                if m == nil {
                        m = &MultiError{}
                }
                m.Add(errors.New("name is nil"))
        }

        return m
}
```

在这个实现中，`m` 被初始化为 `*MultiError` 的零值；因此：m 的值为 nil。如果完整性检查失败，我们会在需要时分配一个新的 `MultiError`，然后追加一个错误。最后，我们返回 `m`，它可以是 nil 指针，也可以是指向 `MultiError` 结构的指针，具体取决于检查。

现在，让我们通过运行一个有效的案例来测试这个实现 `Customer`：

```go
customer := Customer{Age: 33, Name: "John"}
if err := customer.Validate(); err != nil {
        log.Fatalf("customer is invalid: %v", err)
}
```

这里将是输出：

`2021/05/08 13:47:28 customer is invalid: <nil>`

这个结果可能非常令人意外。确实，`Customer` 是有效的，但是 `err != nil` 条件为真，并且记录了打印的错误 `<nil>` 那么，问题是什么？

在 Go 中，我们必须知道指针接收器可以为 nil。让我们通过创建一个虚拟类型并使用 nil 指针接收器调用一个方法来进行实验：

```go
type Foo struct{}
    
func (foo *Foo) Bar() string {
        return "bar"
}

func main() {
        var foo *Foo
        fmt.Println(foo.Bar())
}
```

`foo` 被初始化为指针的零值：nil。然而，这段代码可以编译，如果我们运行它，它会打印 `bar`。nil 指针是一个有效的接收器。

但为什么会这样呢？在 Go 中，方法只是函数的一些语法糖，函数的第一个参数是接收者。因此，我们看到的 `Bar` 方法类似于这个函数：

```go
func Bar(foo *Foo) string { 
        return "bar"
}
```

我们知道将 nil 指针传递给函数是有效的。因此，使用 nil 指针作为接收器也是有效的。

让我们回到我们最初的例子：

```go
func (c Customer) Validate() error {
        var m *MultiError

        if c.Age < 0 {
                // ...
        }
        if c.Name == "" {
                // ...
        }

        return m
}
```

`m` 被初始化为指针的零值：nil。然后，如果所有检查都有效，则提供给 return 语句的 参数不是直接 `nil` 而是一个 `nil` 指针。由于 `nil` 指针是有效的接收器，因此将结果转换为接口不会产生 `nil` 值。换句话说，`Validate` 的调用者总是会得到一个非零错误。

为了明确这一点，让我们记住，在 Go 中，接口是一个调度包装器。在这里，包装器为 nil（`MultiError` 指针），而包装器不是（`error` 接口）：

![](https://img.exciting.net.cn/37.png)

因此，无论 `Customer` 提供了什么，这个函数的调用者总是会收到一个非 nil 错误。理解这种行为是必要的，因为它是一个普遍存在的 Go 错误。

那么，我们应该怎么做才能修复这个例子呢？在函数末尾，最简单的解决方案是仅在 `m` 不为 nil 时才返回：

```go
func (c Customer) Validate() error {
        var m *MultiError

        if c.Age < 0 {
                // ...
        }
        if c.Name == "" {
                // ...
        }

        if m != nil {
                return m
        }
        return nil
}
```

在方法结束时，我们检查 `m` 是否不为 nil。 如果为真，我们返回 `m` ； 否则，我们显式地返回 `nil`。 因此，在有效 `Customer` 的情况下，我们返回一个 `nil` 接口，而不是一个转换为非 `nil` 接口的 `nil` 接收器。

我们在本节中已经看到，在 Go 中，允许使用 nil 接收器，并且从 nil 指针转换的接口不是 nil 接口。出于这个原因，当我们必须返回一个接口时，我们不应该返回一个 nil 指针，而是直接返回一个 nil 值。一般来说，有一个 nil 指针不是一个理想的状态，这意味着一个可能的错误。

我们在本节中看到了一个错误示例，因为它是导致此错误的最常见情况。然而，请注意，这个问题不仅与错误有关。使用指针接收器实现的任何接口都可能发生这种情况。

下一节将讨论使用文件名作为函数输入时的常见设计错误。