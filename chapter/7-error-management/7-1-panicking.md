# 7 错误管理

> **本章概要**
> * 了解何时 panic,了解何时包装错误
> * 或不包装错误自 Go 1.13 以来有效比较错误
> * 类型或错误值学习如何以惯用方式处理错误了解
> * 如何忽略错误处理延迟调用中的错误

错误管理是构建健壮且可读应用程序的一个基本方面，它可能与代码库的任何其他部分一样重要。在 Go 中，错误管理非常单一。事实上，它不像大多数编程语言那样依赖传统的 try/catch 机制。相反，错误作为正常返回值返回。本章将介绍与错误相关的最常见错误。

## 7.1 恐慌(panicking) 

Go 新手对错误处理有些困惑是很常见的。在 Go 中，错误通常由函数或方法管理，返回错误类型作为最后一个参数。然而，人们可能会发现这种方法有些令人惊讶，并且很想在 Java 或 Python 等语言中使用 panic 和 recover 重现异常处理。因此，让我们重新思考一下 panic 的概念，并讨论使用 panic，何时是适当的，何时是不应该的。

在 Go 中，panic 是一个内置函数，可以停止普通流程：

```go
func main() {
    fmt.Println("a")
    panic("foo")
    fmt.Println("b")
}
```

此代码打印 `a` 然后在打印 `b` 之前停止：

```shell
a
panic: foo

goroutine 1 [running]:
main.main()
        main.go:7 +0xb3
```

一旦触发了 panic，它就会继续到调用堆栈，直到当前的 goroutine 返回，或者 `panic` 被 `recover` 捕获：

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recover", r)
        }
    }()

    f()
}

func f() {
    fmt.Println("a")
    panic("foo")
    fmt.Println("b")
}
```

在 `f` 函数中，一旦调用了 `panic`，它就会停止该函数的当前执行并进入调用堆栈： `main`。在 `main` 中，当 panic 被 `recover` 捕获时，它不会停止 goroutine：

```shell
a
    recover foo
```

我们应该注意，调用 `recover()` 来捕获 goroutine 恐慌(panicking)只在 `defer` 函数内部有用；否则，该函数将返回 nil 并且没有其他效果。这是因为当周围的函数 panic 时，也会执行 defer 函数。

既然我们对 panic 有了新的认识，让我们来解决这个问题：什么时候 panic 是合适的？

在 Go 中，panic 用于发出真正异常情况的信号。例如，发出编码级错误的信号。如果我们深入研究 `net/http` 包，我们可以注意到，在 `WriteHeader` 方法中，调用了一个 `checkWriteHeaderCode` 函数来检查状态码是否有效：

```go
func checkWriteHeaderCode(code int) {
    if code < 100 || code > 999 {
        panic(fmt.Sprintf("invalid WriteHeader code %v", code))
    }
}
```

如果状态码无效，这个函数会 panic，这将是一个纯粹的程序错误。

在注册数据库驱动程序时，可以在 `database/sql` 包中找到另一个基于编码错误的示例：

```go
func Register(name string, driver driver.Driver) {
    driversMu.Lock()
    defer driversMu.Unlock()
    if driver == nil {
        panic("sql: Register driver is nil")
    }
    if _, dup := drivers[name]; dup {
        panic("sql: Register called twice for driver " + name)
    }
    drivers[name] = driver
}

```

如果驱动程序为 nil（`driver.Driver` 是一个接口）或已经注册，则此函数会发生 panic。 在这两种情况下，它都会再次被视为编码级错误。此外，在大多数情况下（例如，使用 `go-sql-driver/mysql` 最流行的 Go 的 MySQL 驱动程序），调用 `Register` 是通过一个限制错误处理的 init 函数完成的。由于所有这些原因，一旦出现错误，设计人员就会识别出 panic。

另一个需要 panic 的用例是我们的应用程序需要一个依赖项，但它无法初始化它。例如，假设我们公开了一项服务来创建新的客户帐户。此服务在某个阶段需要验证提供的电子邮件地址。为了实现它，我们可以决定使用正则表达式。

在 Go 中，`regexp` 包公开了两个函数来从字符串创建正则表达式：`Compile` 和 `MustCompile`。前者返回一个 `*regexp.Regexp` 和一个错误，而后者只返回一个 `*regexp.Regexp` 但在出现错误时会触发 panic。在这种情况下，正则表达式是强制依赖项。事实上，如果我们编译失败，我们将永远无法验证任何电子邮件输入。因此，我们可能倾向于使用 `MustCompile` 和 `panic` 以防出错。

应该谨慎使用 Go 中的 panic。我们已经看到了两个突出的案例，一个是编码级错误的信号，另一个是我们的应用程序无法创建强制依赖项。因此，导致应用程序停止的异常情况。在大多数其他情况下，错误管理应该通过返回正确错误类型作为最后一个返回参数的函数来完成。

现在让我们开始讨论错误，看看何时包装错误。