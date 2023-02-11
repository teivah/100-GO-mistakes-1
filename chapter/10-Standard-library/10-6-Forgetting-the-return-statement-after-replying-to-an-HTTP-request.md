## 10.6 回复 HTTP 请求后忘记 return 语句

在编写 HTTP 处理程序时，很容易在回复 HTTP 请求后忘记返回语句。这可能会导致一种奇怪的情况，我们应该在发生错误后停止处理程序，但事实上，我们没有。

我们可以在下面的例子中观察到这种情况：

```go
func handler(w http.ResponseWriter, req *http.Request) {
    err := foo(req)
    if err != nil {
        http.Error(w, "foo", http.StatusInternalServerError)
    }

    // ...
}
```
如果 `foo` 返回错误，我们使用 `http.Error` 来处理它，它使用 `foo` 错误消息和 500 Internal Server Error 回复请求。这段代码的问题在于，如果我们进入 `if err != nil` 分支，应用程序将继续执行，因为 `http.Error` 不会停止处理程序的执行。

这种错误的真正影响是什么？首先，让我们以 HTTP 级别进行讨论。例如，如果我们通过添加一个步骤来编写成功的 HTTP 响应正文和状态代码来完成前面的 HTTP 处理程序：

```go
func handler(w http.ResponseWriter, req *http.Request) {
    err := foo(req)
    if err != nil {
        http.Error(w, "foo", http.StatusInternalServerError)
    }

    _, _ = w.Write([]byte("all good"))
    w.WriteHeader(http.StatusCreated)
}
```

在这种情况下 `err != nil`，HTTP 响应如下：

```shell
foo
all good
```

因此，响应将包含错误消息和成功消息。关于 HTTP 状态码，我们只返回第一个。在前面的示例中：500。但是，Go 也会记录一个警告：

```shell
2021/10/29 16:45:33 http: superfluous response.WriteHeader call from main.handler (main.go:20)
```

这个警告意味着我们多次尝试写入状态码，这是多余的。

执行方面，主要影响是继续执行应该停止的功能。例如，如果 `foo` 在错误旁边返回一个指针，继续执行将意味着使用这个指针并可能导致 nil 指针取消引用（goroutine panic）。

解决这个错误的方法是继续考虑在 `http.Error` 之后添加 `return` 语句：

```go
func handler(w http.ResponseWriter, req *http.Request) {
    err := foo(req)
    if err != nil {
        http.Error(w, "foo", http.StatusInternalServerError)
        return
    }

    // ...
}
```

多亏了 `return` 语句，如果我们在 `if err != nil` 分支中返回，函数将停止执行。

这个错误可能不是本书中最复杂的。然而，很容易忘记它，以至于这个错误相当频繁。让我们永远记住 `http.Error` 不会停止处理程序的执行，并且必须手动添加。最后但并非最不重要的一点是，这样的问题可以而且应该在测试期间以良好的覆盖率被发现。

本章的最后一节将继续讨论 HTTP。我们将介绍为什么生产级应用程序不应依赖默认的 HTTP 客户端和服务器实现。