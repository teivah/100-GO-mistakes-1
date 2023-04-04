## 2.11 不使用函数式选项模式

在设计 API 时，可能会出现一个问题：如何处理可选配置？有效解决这个问题可以提高我们的 API 的便利性。本节将介绍一个具体的例子，并介绍处理它的不同方法。

在本例中，假设我们必须设计一个公开函数以创建 HTTP 服务器的库。此函数将接受不同的输入：地址和端口。该功能的骨架如下：

```go
func NewServer(addr string, port int) (*http.Server, error) {
    // ...
}
```

我们 client 的库已经开始使用这个函数，每个人都很高兴。然而，在某个时候，client 开始抱怨此功能有些有限，缺乏其他参数（例如，写入超时、连接上下文）。然而，我们注意到，添加新的函数参数将破坏兼容性，迫使 client 修改他们调用 `NewServer` 的方式。

与此同时，我们希望通过这种方式丰富与端口管理相关的逻辑：

* 如果未设置端口，则使用默认端口
* 如果端口为负数，则返回错误
* 如果端口等于0，则使用随机端口
* 否则，它将使用client提供的端口

![](https://img.exciting.net.cn/4.png)

我们如何以 API 友好的方式实现此功能？让我们看看不同的选项。

### 2.11.1 配置用的结构体

由于 Go 不支持函数签名中的可选参数，第一个可能的方法是使用配置结构来传达什么是强制性和可选的。例如，强制性参数可以作为函数参数存在，而可选参数可以在配置结构中处理：

```go
type Config struct {
    Port  int
}

func NewServer(addr string, cfg Config) {
}
```

此解决方案解决了兼容性问题。事实上，如果我们添加新选项，它不会在调用方崩溃。然而，这种方法并不能解决我们与端口管理相关的要求。事实上，我们应该记住，如果没有提供结构字段，它将初始化为零值：

* 0是整数类型
* 0.0是浮点类型
* `""` 是字符串类型
* nil 可能是切片、map、通道、指针、接口和函数

因此，在以下示例中，两个结构是相等的：

```go
c1 := httplib.Config{
    Port: 0,
}
c2 := httplib.Config{

}
```

就我们而言，我们需要找到一种方法来区分故意设置为0的端口和缺失的端口。也许一种选择可能是以这种方式将配置结构的所有参数作为指针处理：

```go
type Config struct {
    Port  *int
}
```

使用整数的指针，在语义上，我们可以区分值为零和缺失的值（零指针）。

这个选项会起作用，但有几个缺点。首先，client 提供整数指针并不方便。client 必须创建一个变量，然后以这种方式传递指针：

```go
port := 0
config := httplib.Config{
    Port: &port,
}
```

它不是这样的展示品，但整个API变得不那么方便了。此外，我们添加的选项越多，代码就越复杂。

第二个缺点是，使用我们库进行默认配置的 client 必须以这种方式传递一个空结构：

```go
httplib.NewServer("localhost", httplib.Config{})
```

这个代码看起来不太好。读者必须了解这个魔法结构的含义。

另一种选择是使用下一节中介绍的经典构建器模式。

### 2.11.2 构造器模式

最初是四人帮设计模式的一部分，构建者模式为各种对象创建问题提供了灵活的解决方案。`Config` 的构造与结构本身分离。它需要一个额外的结构（`ConfigBuilder`），该结构将接收配置和构建 `Config` 的方法。

让我们看看一个具体的例子，以及它如何帮助我们设计一个友好的 API 来满足我们的所有要求，包括端口管理：

```go
type Config struct {
    Port int
}

type ConfigBuilder struct {
    port *int
}

func (b *ConfigBuilder) Port(port int) *ConfigBuilder {
    b.port = &port
    return b
}

func (b *ConfigBuilder) Build() (Config, error) {
    cfg := Config{}

    if b.port == nil {
        cfg.Port = defaultHTTPPort
    } else {
        if *b.port == 0 {
            cfg.Port = randomPort()
        } else if *b.port < 0 {
            return Config{}, errors.New("port should be positive")
        } else {
            cfg.Port = *b.port
        }
    }
    return cfg, nil
}

func NewServer(addr string, config Config) (*http.Server, error) {
    // ...
}
```

`ConfigBuilder` 结构保存 client 配置。它公开了设置端口的 `Port` 方法。通常，这样的配置方法返回构建器本身，以便我们可以使用方法链（例如，`builder.Foo("foo").Bar("bar")`）。它还公开了一个 `Build` 方法，该方法保存初始化端口值时的逻辑（无论指针是否为零等），然后在创建后返回 `Config` 结构。

> **Note** 构建器模式不可能单独实现。例如，人们可能更喜欢一种方法，即定义最终端口值的逻辑在 `Port` 方法中，而不是 `Build`。然而，这些章节的范围是概述构建者模式，而不是深入研究所有不同的可能变化。

然后，client 将以以下方式使用我们基于构建器的 API（我们假设我们已经将代码放在 `httplib` 包中）：

```go
builder := httplib.ConfigBuilder{}
builder.Port(8080)
cfg, err := builder.Build()
if err != nil {
    return err
}

server, err := httplib.NewServer("localhost", cfg)
if err != nil {
    return err
}
```

首先，客户端创建一个 `ConfigBuilder`，并使用它来设置可选字段，如端口。然后，它调用 Build 方法并检查是否有错误。如果可以，配置将传递给 NewServer。

这种方法使港口管理更加方便。不需要传递整数指针，因为端口方法接受整数。但是，仍然需要传递一个配置结构，如果 client 想要使用默认配置，该结构可能是空的：

`server, err := httplib.NewServer("localhost", nil)`

在某些情况下，另一个缺点与错误管理有关。在抛出异常的编程语言中，如果输入无效，`Port` 等构建器方法可以引发异常。如果我们想保持链式调用的能力，该函数无法返回错误。因此，我们必须推迟 `Build` 方法中的验证。如果 client 可以传递多个选项，但如果端口无效，则希望精确处理该情况，则会使错误处理变得更加复杂。

现在让我们看看另一种方法，称为依赖于变量参数的功能选项模式。

## 2.11.3 函数选项模式

我们将讨论的最新方法是函数选项模式。虽然有不同的实现，但略有变化，但主要想法如下：

* 未导出的结构包含配置：`options` 。
* 每个选项都是返回相同类型的函数：`type Option func(options *options) error`。例如，`WithPort` 接受代表端口的 `int` 参数，并返回表示如何更新 `Option` 结构的 `Option` 类型。

![](https://img.exciting.net.cn/5.png)

以下是 `options` 结构、`Option` 类型和 `WithPort` 选项的 Go 实现。

```go
type options struct {
    port *int
}

type Option func(options *options) error

func WithPort(port int) Option {
    return func(options *options) error {
        if port < 0 {
            return errors.New("port should be positive")
        }
        options.port = &port
        return nil
    }
}
```

在这里，`WithPort` 返回一个闭包。闭包是一个匿名函数，从其主体外部引用变量；在这里，`port` 变量。闭包尊重 `Option` 类型，并实现端口验证逻辑。

每个配置字段都需要创建一个公共函数（以约定的 `With` 前缀开头），其中包含类似的逻辑：根据需要验证输入并更新配置结构。

现在，让我们深入研究提供商方面的最后一部分，`NewServer` 实现。这些选项将作为变量参数传递。因此，我们必须迭代这些选项来突变 `options` 配置结构。

```go
func NewServer(addr string, opts ...Option) (*http.Server, error) {
    var options options
    for _, opt := range opts {
        err := opt(&options)
        if err != nil {
            return nil, err
        }
    }

    // At this stage, the options struct is built and contains the config
    // Therefore, we can implement our logic related to port configuration
    var port int
    if options.port == nil {
        port = defaultHTTPPort
    } else {
        if *options.port == 0 {
            port = randomPort()
        } else {
            port = *options.port
        }
    }

    // ...
}
```

我们首先创建一个空的 `options` 结构。然后，我们迭代每个 `Option` 参数并执行它们（请记住 `Option` 类型是一个函数），以突变 `options` 结构。然后，一旦构建了 `options` 结构，我们就可以实现有关端口管理的最终逻辑。

由于 `NewServer` 接受各种 `Option` 参数，client 现在可以通过在强制地址参数之后传递多个选项来调用此 API。例如：

```go
server, err := httplib.NewServer("localhost",
    httplib.WithPort(8080),
    httplib.WithTimeout(time.Second))
```

但是，如果 client 需要默认配置，它不必提供任何参数，例如我们在之前的方法中看到的空结构：

```go
server, err := httplib.NewServer("localhost")
```

这种模式是功能选项模式。它提供了一种方便且API友好的处理选项的方式。虽然构建器模式可以是一个有效的选项，但它有一些小的缺点，往往使功能选项模式成为在 Go 中解决这个问题的惯用方式。我们还请注意，此模式用于许多不同的 Go 库，例如 gRPC。

下一节将深入研究一个常见的错误：Go 项目组织不善。