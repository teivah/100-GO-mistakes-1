## 11.7 不使用测试工具包

标准库提供了用于测试的实用程序包。一个常见的错误是不了解这些软件包并试图重新发明轮子或依赖其他不太方便的解决方案。本节将深入研究其中的两个包，一个在使用 HTTP 时为我们提 供帮助，另一个在执行 I/O 以及使用 reader 和 writers 时提供帮助。

### 11.7.1 `httptest`

`httptest` 软件包为客户端和服务器的 HTTP 测试提供了实用程序。让我们深入研究这两个用例。

首先，让我们看看 `httptest` 如何在编写 HTTP 服务器时帮助我们。我们将实现一个处理程序来执行一些基本操作：编写标题、正文并返回特定的状态码。为了清楚起见，我们将省略错误处理：

```go
func Handler(w http.ResponseWriter, r *http.Request) {
    w.Header().Add("X-API-VERSION", "1.0")
    b, _ := io.ReadAll(r.Body)
    _, _ = w.Write(append([]byte("hello "), b...))
    w.WriteHeader(http.StatusCreated)
}
```

HTTP 处理程序接受两个参数：请求和写入响应的方式。`httptest` 包为两者提供了实用程序。对于请求，我们可以使用 `httptest.NewRequest` 使用 HTTP 方法、URL 和 正文构建 `*http.Request`。关于响应，我们可以使用 `httptest.NewRecorder` 来记录处理程序中发生的变化。让我们编写这个处理程序的单元测试：

```go
func TestHandler(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "http://localhost",
        strings.NewReader("foo"))
    w := httptest.NewRecorder()
    Handler(w, req)

    if got := w.Result().Header.Get("X-API-VERSION"); got != "1.0" {
        t.Errorf("api version: expected 1.0, got %s", got)
    }

    body, _ := ioutil.ReadAll(w.Body)
    if got := string(body); got != "hello foo" {
        t.Errorf("body: expected hello foo, got %s", got)
    }

    if http.StatusOK != w.Result().StatusCode {
        t.FailNow()
    }
}
```

使用 `httptest` 测试处理程序不会测试传输（HTTP 部分）。测试的重点是直接用请求和记录响应的方式调用处理程序。然后，使用响应记录器，我们编写断言来验证 HTTP 标头、正文和状态代码。

当我们想要测试 HTTP 客户端时，让我们看看硬币的另一面。

我们将编写一个客户端来负责查询一个 HTTP 端点，该端点计算从一个坐标驱动到另一个坐标需 要多长时间。客户端看起来像这样：

```go
func (c DurationClient) GetDuration(url string,
    lat1, lng1, lat2, lng2 float64) (
    time.Duration, error) {
    resp, err := c.client.Post(
        url, "application/json",
        buildRequestBody(lat1, lng1, lat2, lng2),
    )
    if err != nil {
        return 0, err
    }

    return parseResponseBody(resp.Body)
}
```

它对提供的 URL 执行 HTTP POST 请求并返回解析的响应（比如说，一些 JSON）。

如果我们想测试这个客户端怎么办？一种选择是使用 Docker 并启动一个模拟服务器以返回一些预先注册的响应。但是，这种方法会使测试执行缓慢。另一种选择是使用 `httptest.NewServer` 基于我们将提供的处理程序创建本地 HTTP 服务器。服务器启动并运行后，我们将使用其 URL 并将其传递给 `GetDuration`：

```go
func TestDurationClientGet(t *testing.T) {
    srv := httptest.NewServer(
        http.HandlerFunc(
            func(w http.ResponseWriter, r *http.Request) {
                _, _ = w.Write([]byte(`{"duration": 314}`))
            },
        ),
    )
    defer srv.Close()

    client := NewDurationClient()
    duration, err :=
        client.GetDuration(srv.URL, 51.551261, -0.1221146, 51.57, -0.13)
    if err != nil {
        t.Fatal(err)
    }

    if duration != 314*time.Second {
        t.Errorf("expected 314 seconds, got %v", duration)
    }
}
```

在此测试中，我们创建了一个带有返回 `314` 秒的静态处理程序的服务器。也可以根据发送的请求做出断言。此外，当我们调用 `GetDuration` 时，我们提供启动的服务器的 URL。与测试处理程序相比，此测试确实执行了实际的 HTTP 调用，但执行时间仅为几毫秒。

也可以使用带有 `httptest.NewTLSServer` 的 TLS 启动一个新服务器，并使用 `httptest.NewUnstartedServer` 创建一个未启动的服务器，这样我们就可以懒惰地启动它。

让我们记住在 HTTP 应用程序上下文中工作时 `httptest` 有多大帮助。无论是在编写服务器还是客户端时，`httptest` 都可以帮助我们创建高效的测试。

### 11.7.2 `iotest`

`iotest` 包实现了用于测试读取器和写入器的实用程序。这是 Go 开发人员经常忘记的一个方便的包。

首先，在实现自定义 `io.Reader` 时，我们应该记住使用 `iotest.TestReader` 对其进行测试。此实用程序函数测试读取器的行为是否正确：它正确返回读取的字节数、填充提供的切片等。如果提供者读取器实现诸如 `io.ReaderAt` 之类的接口，它还会测试不同的行为。

让我们假设一个自定义的 `LowerCaseReader` 从给定的输入 `io.Reader` 流式传输小写字母。以下是测试该阅读器是否行为不端的方法：

```go
func TestLowerCaseReader(t *testing.T) {
    err := iotest.TestReader(
        &LowerCaseReader{reader: strings.NewReader("aBcDeFgHiJ")},
        []byte("acegi"),
    )
    if err != nil {
        t.Fatal(err)
    }
}
```

我们通过提供自定义的 `LowerCaseReader` 和期望，小写字母：`acegi` 来调用 `io.TestReader`。

使用 `iotest` 包的另一个用例是确保使用读取器和写入器的应用程序容错：

* `iotest.ErrReader`：创建一个返回提供的错误的 `io.Reader`
* `iotest.HalfReader`：创建一个 `io.Reader`，它只从 `io.Reader` 中读取请求字节数的一半
* `iotest.OneByteReader`：创建一个 `io.Reader`，它从 `io.Reader` 中为每次非空读取读取一个字节
*`iotest.TimeoutReader`：创建一个 `io.Reader`，它在没有数据的情况下第二次读取时返回错误，但后续调用会成功
*`iotest.TruncateWriter`：创建一个 `io.Writer` 写入 `io.Writer` 但在 n 字节后静默停止

例如，假设我们实现了以下从读取器读取所有字节开始的函数：

```go
func foo(r io.Reader) error {
    b, err := io.ReadAll(r)
    if err != nil {
        return err
    }
    
    // ...
}
```

我们希望确保我们的函数具有弹性，例如，如果提供的读取器在读取 期间失败（例如，模拟网络错误）：

```go
func TestFoo(t *testing.T) {
    err := foo(iotest.TimeoutReader(
        strings.NewReader(randomString(1024)),
    ))
    if err != nil {
        t.Fatal(err)
    }
}
```

在这里，我们使用 `io.TimeoutReader` 包装了一个 `io.Reader`。正如我们提到的，第二次读取将失败。如果我们运行这个测试来确保我们的函数容错，我们将得到一个测试失败。 事实上，`io.ReadAll` 会返回它发现的任何错误。 

知道了这一点，我们就可以实现我们的自定义 `readAll` 函数，它最多可以容忍 n 个错误：

```go
func readAll(r io.Reader, retries int) ([]byte, error) {
    b := make([]byte, 0, 512)
    for {
        if len(b) == cap(b) {
            b = append(b, 0)[:len(b)]
        }
        n, err := r.Read(b[len(b):cap(b)])
        b = b[:len(b)+n]
        if err != nil {
            if err == io.EOF {
                return b, nil
            }
            retries--
            if retries < 0 {
                return b, err
            }
        }
    }
}
```

此实现类似于 `io.ReadAll`，但它还处理可配置的重试。如果我们将初始函数的实现更改为使用自定义的 `readAll` 而不是 `io.ReadAll`，测试将不会再失败：

```go
func foo(r io.Reader) error {
    b, err := readAll(r, 3)
    if err != nil {
        return err
    }

    // ...
}
```

我们已经看到了一个示例，说明如何在从 `io.Reader` 读取数据时检查函数是否容错。 我们依靠 `iotest` 包进行了测试。

在进行 I/O 并使用 `io.Reader` 和 `io.Writer` 时，让我们记住 `iotest` 包是多么方便。正如我们所见，它提供了实用程序，包括一些用于测试自定义 `io.Reader` 行为的实用程序，以及测试我们的应用程序在读取或写入数据时是否发生错误的功能。

下一节将讨论一些可能导致编写不准确基准的常见陷阱。