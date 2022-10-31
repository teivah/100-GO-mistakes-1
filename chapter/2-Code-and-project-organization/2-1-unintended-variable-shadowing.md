# 2 代码和项目组织

> **本章概要**
* 习惯性地组织我们的代码
* 高效处理抽象：接口和泛型 
* 关于如何构建项目的最佳实践

以干净、惯用和可维护的方式组织Go代码库并非易事。理解与代码和项目组织相关的所有最佳实践需要经验和错误。例如，要避免哪些陷阱，例如变量遮蔽和嵌套代码滥用，如何构建包，何时何地使用接口或泛型，init函数，实用程序包？在本章中，我们将深入研究常见的组织错误。

## 2.1 意外的阴影变量

变量的范围是指变量可以被引用的地方。换句话说，名称绑定有效的应用程序部分。在Go中，在块中声明的变量名可以在内部块中重新声明。这种称为变量阴影的原理很容易出现常见错误。

在下面的示例中，我们将看到由于阴影变量而产生的意外副作用。我们将以两种不同的方式创建HTTP客户端，具体取决于 `tracing` 布尔值的值：

```go
var client *http.Client
if tracing {
        client, err := createClientWithTracing()
        if err != nil {
                return err
        }
        log.Println(client)
} else {
        client, err := createDefaultClient()
        if err != nil {
                return err
        }
        log.Println(client)
}
// Use client
```

首先，我们声明一个 `client` 变量。然后，我们在两个内部块中使用短变量声明运算符(`:=`)将其分配给内部 `client` 变量，而不是外部变量。因此，按照这个示例，外部变量将始终为 nil。

> **Note** 此代码可以编译，因为在日志调用中使用了内部 `client` 变量。如果没有，我们会出现编译错误：`client declared and not used`。

我们如何确保为原始 `client` 变量赋值？有两种不同的选择。

第一种选择是通过这种方式在内部块中使用临时变量：

```go
var client *http.Client
if tracing {
        c, err := createClientWithTracing()
        if err != nil {
                return err
        }
        client = c
} else {
        // Same logic
}
```

在这里，我们将结果分配给一个临时 `c` 变量，其范围仅在 `if` 块内。然后，我们将其分配回 `client` 变量。同时，我们对 `else` 部分做同样的事情。

第二种选择是使用内部块中的赋值运算符(`=`)将函数结果直接分配给 `client` 变量。但是，它需要创建一个 `error` 变量，因为赋值运算符仅在已声明变量名称时才起作用：

```go
var client *http.Client
var err error
if tracing {
        client, err = createClientWithTracing()
        if err != nil {
                return err
        }
} else {
        // Same logic
}
```

我们可以直接将其分配给临时变量，而不是先分配给一个临时变量 `client`。

这两个选项都是完全有效的。主要区别在于我们在第二个选项中只执行了一个赋值，这可能被认为更容易阅读。此外，使用第二个选项，可以在 `if/else` 语句之外实现错误处理：

```go
if tracing {
        client, err = createClientWithTracing()
} else {
        client, err = createDefaultClient()
}
if err != nil {
        // Common error handling
}
```

当在内部块中重新声明变量名称时，就会发生变量阴影，我们已经看到这种做法很容易出错。施加规则以禁止阴影变量取决于我们的个人品味。例如，有时，重用现有变量名会非常方便，例如 `err` 表示错误。然而，总的来说，我们应该保持谨慎，因为我们已经看到我们可能会遇到这样一种情况：代码编译，但接收到值的变量不是预期的。在本章后面，我们还将看到如何检测阴影变量，这可能有助于我们发现可能的错误。

以下部分将了解为什么避免滥用嵌套代码很重要。