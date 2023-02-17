## 10.7 不要使用默认的 HTTP 客户端和服务器
`http` 包提供 HTTP 客户端和服务器实现。然而，开发人员很容易并且经常犯一个常见错误：在最终部署到生产环境的应用程序上下文中依赖默认实现。让我们了解问题是什么以及如何克服它们。

### 10.7.1 HTTP 客户端

首先，让我们定义默认客户端的含义；我们将以 GET 请求为例。这意味着要么像这样使用 `http.Client` 结构的零值：

```go
client := &http.Client{}
resp, err := client.Get("https://golang.org/")
```

或者，使用 `http.Get` 函数：

```go
resp, err := http.Get("https://golang.org/")
```

最后，这两种方法都与 `http.Get` 函数使用 `http.DefaultClient` 相同，这也是基于 `http.Client` 的零值：

```go
// DefaultClient is the default Client and is used by Get, Head, and Post.
var DefaultClient = &Client{}
```

那么，使用默认的 HTTP 客户端有什么问题呢？

首先，默认客户端不指定任何超时。这种没有超时的情况可能不是我们想要的生产级系统，它会导致许多问题，例如可能耗尽系统资源的永无止境的请求。

在深入研究发出请求时的可用超时之前，让我们回顾一下 HTTP 请求中涉及的五个步骤：

1. 拨号建立 TCP 连接
2. TLS 握手（如果启用）
3. 发送请求
4. 读取响应头 
5. 读取响应体

以下是这些步骤与主要客户端超时的关系：

![](https://img.exciting.net.cn/71.png)

四个主要的超时时间如下：

* `net.Dialer.Timeout`：指定拨号等待连接完成的最长时间。
* `http.Transport.TLSHandshakeTimeout`：指定等待 TLS 握手的最长时间。
* `http.Transport.ResponseHeaderTimeout`：指定等待服务器响应标头的时间。 
* `http.Client.Timeout`：指定请求的时间限制。它包括所有步骤，从第一步（拨号）到第五步（阅读响应正文）。

> **Note** 此错误表示端点未能按时响应。我们收到有关标头的此错误，因为读取它们是等待响应的第一步。
> 
> 如果指定 `http.Client.Timeout`，您可能已经遇到以下错误：
> 
> `net/http: request canceled (Client.Timeout exceeded while awaiting headers)`

我们为拨号、TLS 握手和读取响应标头创建了一个具有一秒超时的客户端。同时，每个请求都会有一个全局的 5 秒超时。

使用默认 HTTP 客户端要记住的第二个方面是如何处理连接。

默认情况下，HTTP 客户端执行连接池。实际上，默认客户端重用连接（可以通过将 `http.Transport.DisableKeepAlives` 设置为 true 来禁用它）。有一个额外的超时来指定空闲连接在池中保留多长时间：`http.Transport.IdleConnTimeout`。默认值为 90 秒，这意味着连接可以在这段时间内被其他请求重用。在那之后，如果连接没有被重用，它将被关闭。

要配置池中的连接数，我们必须覆盖 `http.Transport.MaxIdleConns`。该值默认设置为 100。但是，有一点需要注意：每个主机存在 `http.Transport.MaxIdleConnsPerHost` 限制，默认设置为 2。例如，如果我们触发 100 个请求同一台主机，之后连接池中只会保留 2 个连接。因此，如果我们再次触发 100 个请求，我们将不得不重新打开至少 98 个连接。这也是一个需要注意的重要配置，因为如果我们必须处理对同一主机的大量并行请求，它会影响平均延迟。 

让我们记住，对于生产级系统，我们可能想要覆盖默认超时。同时，调整与连接池相关的参数也会对延迟产生重大影响。

### 10.7.2 HTTP 服务端

在实现 HTTP 服务器时，我们也应该小心。同样，可以使用 `http.Server` 的零值创建默认客户端：

```go
server := &http.Server{}
server.Serve(listener)
```

或者，使用 `http.Serve`、`http.ListenAndServe` 或 `http.ListenAndServeTLS` 等函数，因为它们也依赖于默认的 `http.Server`。

接受连接后，HTTP 响应分为五个步骤：

1. 等待客户端发送请求 
2. TLS 握手（如果启用）
3. 读取请求头 
4. 读取请求正文 
5. 写入响应

> **Note** 不必对已建立的连接重复 TLS 握手。

以下是这些步骤与主服务器超时的关系：

![](https://img.exciting.net.cn/72.png)

三个主要的超时时间如下：

* `http.Server.ReadHeaderTimeout`：一个字段，用于指定读取请求标头的最长时间。 
* `http.Server.ReadTimeout`：指定读取整个请求的最长时间的字段
* `http.TimeoutHandler`：一个包装函数来指定处理程序完成的最长时间。

最后一个参数不是服务器参数，而是处理程序顶部的包装器以限制其持续时间。如果处理程序未能按时响应，服务器将使用特定消息回复 _503 Service Unavailable_。同时，传递给处理程序的上下文将被取消。

> **Note** 我们故意省略了 `http.Server.WriteTimeout`，因为 `http.TimeoutHandler` 已经发布（Go 1.8），这不是必需的。事实上，`http.Server.WriteTimeout` 有一些问题。首先，它的行为取决于是否启用了 TLS，这使得它的理解和使用更加复杂。此外，如果达到超时，它会关闭 TCP 连接而不返回正确的 HTTP 代码。同时，它不会将取消传播到处理程序上下文。因此，处理程序可以在不知道 TCP 连接已经关闭的情况下继续执行。

在将我们的接口暴露给不受信任的客户端时，最佳实践可能是至少设置 `http.Server.ReadHeaderTimeout` 字段并使用 `http.TimeoutHandler` 包装函数。否则，例如，客户端可能会利用它并创建永无止境的连接，从而导致系统资源耗尽。

以下是设置具有这些超时的服务器的方法：

```go
s := &http.Server{
    Addr:              ":8080",
    ReadHeaderTimeout: 500 * time.Millisecond,
    ReadTimeout:       500 * time.Millisecond,
    Handler:           http.TimeoutHandler(handler, time.Second, "foo"),
}
```

`http.TimeoutHandler` 包装提供的处理程序。在这里，如果 `handler` 程序在一秒钟内没有响应，它会返回 503 并以 `foo` 作为 HTTP 响应。

与我们描述的关于 HTTP 客户端的方式相同，我们还可以在服务器端配置启用 keep‑alives 时下一个请求的最长时间；可以使用`http.Server.IdleTimeout`:

```go
s := &http.Server{
    // ...
    IdleTimeout: time.Second,
}
```

请注意，如果未设置 `http.Server.IdleTimeout`，则使用 `http.Server.ReadTimeout` 的值作为空闲超时。如果两者都没有设置，则不会有任何超时，并且连接将保持打开状态，直到它们被客户端关闭。

对于生产级应用程序，我们需要确保不使用默认的 HTTP 客户端和服务器。否则，由于没有超时，甚至恶意客户端利用我们的服务器没有任何超时的事实，它可能导致请求永远卡住。

