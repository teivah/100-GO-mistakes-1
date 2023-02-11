## 10.3 JSON处理常见错误

Go 通过 `encoding/json` 包对 JSON 有很好的支持。本节将介绍与编码（编组）和解码（解组） JSON 数据相关的三个常见错误。

### 10.3.1 类型嵌入导致的意外行为

在 *未意识到类型嵌入可能存在的问题* 中，我们解释了与类型嵌入相关的可能问题。在 JSON 处理的上下文中，让我们讨论另一个可能导致意外封送/解封结果的潜在影响。

在以下示例中，我们将创建一个包含 ID 和嵌入时间戳的 `Event` 结构：

```go
type Event struct {
    ID int
    time.Time
}
```

在这里，由于 `time.Time` 是嵌入的，与我们之前描述的方式相同，我们可以直接在 `Event` 级别访问 `time.Time` 方法；例如， `event.Second()`。

使用 JSON 封送处理的嵌入式字段可能产生哪些影响？让我们在下面的例子中找到它。我们将实例化一个 `Event` 并将其编组为 JSON。这段代码的输出应该是什么？

```go
event := Event{
    ID:   1234,
    Time: time.Now(),
}

b, err := json.Marshal(event)
if err != nil {
    return err
}

fmt.Println(string(b))
```

我们可能期望此代码打印以下内容:

```shell
{"ID":1234,"Time":"2021-04-19T21:15:08.381652+02:00"}
```

相反，它将打印：

```shell
"2020-12-21T00:08:22.81013+01:00"
```

我们如何解释这个输出？`ID` 字段和 1234 值发生了什么变化？由于该字段被导出，它应该已经被封送。要理解这个问题，我们必须强调两点。

首先，正如不知道类型嵌入可能存在的问题中所讨论的，如果嵌入字段类型实现了接口，则包含嵌入字段的结构也将实现该接口。

其次，我们可以通过使类型实现 `json.Marshaler` 接口来更改默认的封送处理行为。此接口包含一个 `MarshalJSON` 函数：

```go
type Marshaler interface {
    MarshalJSON() ([]byte, error)
}
```

这是一个自定义编组的示例：

```go
type foo struct{}

func (foo) MarshalJSON() ([]byte, error) {
    return []byte(`"foo"`), nil
}

func main() {
    b, err := json.Marshal(foo{})
    if err != nil {
        panic(err)
    }
    fmt.Println(string(b))
}
```

由于我们通过实现 `Marshaler` 接口更改了默认 JSON 编组行为，因此此代码将打印 `"foo"`。

弄清楚这两点后，让我们回到 `Event` 结构的最初问题：

```go
type Event struct {
    ID int
    time.Time
}
```

我们要知道 `time.Time` **实现** 了 `json.Marshaler` 接口。由于 `time.Time` 是 `Event` 的嵌入式字段，因此它提升了它的方法。 因此，`Event` 也实现了 `json.Marshaler`。

因此，将 `Event` 传递给 `json.Marshal` 将不会使用默认的封送处理行为，而是使用 `time.Time` 提供的封送处理行为。这就是封送 `Event` 导致忽略 `ID` 字段的原因。

> **Note** 如果我们使用 `json.Unmarshal` 解组 `Event`，我们也会面临相反的问题。

要解决此问题，有两种主要可能性。

首先，我们可以通过添加名称使 `time.Time` 字段不再嵌入：

```go
type Event struct {
    ID int
    Time time.Time
}
```

这样，如果我们编组这个 `Event` 结构的一个版本，它将打印如下内容：

```shell
{"ID":1234,"Time":"2020-12-21T00:30:41.413417+01:00"}
```

如果我们想保留或必须保留嵌入的 `time.Time` 字段，另一种选择是让 `Event` 实现 `json.Marshaler` 接口：

```go
func (e Event) MarshalJSON() ([]byte, error) {
    return json.Marshal(
        struct {
            ID   int
            Time time.Time
        }{
            ID:   e.ID,
            Time: e.Time,
        },
    )
}
```

在此解决方案中，我们在定义反映 `Event` 结构的匿名结构时实现了自定义 `MarshalJSON` 方法。然而，此解决方案比较麻烦，并且需要确保 `MarshalJSON` 方法始终与 `Event` 结构保持同步。

我们应该小心嵌入字段。虽然提升嵌入字段类型的字段和方法有时很方便，但也可能导致细微的错误，因为它可以使父结构在没有明确信号的情况下实现某些接口。同样，在使用嵌入式字段时，我们应该清楚地了解可能的副作用。

在下一节中，我们将看到另一个与使用 `time.Time` 相关的常见 JSON 错误。

### 10.3.2 JSON 和单调时钟

当封送或取消封送包含 `time.Time` 类型的结构时，我们有时会遇到意想不到的比较错误。 深入研究 `time.Time` 有助于完善我们的假设并防止可能的错误。

操作系统处理两种不同的时钟类型：壁式时钟和单调时钟。本节将首先深入研究 这两种时钟类型，然后查看使用 JSON 和 `time.Time` 可能产生的影响。

挂钟用于了解一天中的当前时间。此时钟可能会发生变化。例如，如果使用 NTP（网络时间协议）同步，时钟可以在时间上向后或向前跳跃。我们不应该使用挂钟来测量持续时间，因为我们可能会遇到奇怪的行为，例如负持续时间。这就是操作系统提供第二种时钟类型的原因：单调时钟。单调时钟保证时间永远向前移动，不受时间跳跃的影响。它可能会受到潜在频率调整的影响（例如，如果服务器检测到本地石英的移动速度与 NTP 服务器不同），但不会受到时间跳跃的影响。

在下面的示例中，我们将继续考虑一个 `Event` 结构，但包含一个 `time.Time` 字段（未嵌入）：

```go
type Event struct {
	Time time.Time
}
```

我们将实例化一个 `Event`，将其编组为 JSON 并将其解编为另一个结构。然后，我们将比较这两个结构。让我们看看编组/解组过程是否总是对称的：

```go
t := time.Now()
event1 := Event{
    Time: t,
}

b, err := json.Marshal(event1)
if err != nil {
    return err
}

var event2 Event
err = json.Unmarshal(b, &event2)
if err != nil {
    return err
}

fmt.Println(event1 == event2)
```

这段代码的输出应该是什么？它将打印错误，而不是真实。我们该如何解释呢？

首先，让我们打印 `event1` 和 `event2` 的内容：

```go
fmt.Println(event1.Time)
fmt.Println(event2.Time)
```

```shell
2021-01-10 17:13:08.852061 +0100 CET m=+0.000338660
2021-01-10 17:13:08.852061 +0100 CET
```

所以我们可以注意到它打印了两个不同的内容。`event1` 与 `event2` 类似，除了 `m=+0.000338660` 部分。它的意义是什么？

在 Go 中，不是将两个时钟拆分为两个不同的 API，而是 `time.Time` 可以同时包含墙壁时间和单调时间。当我们使用 `time.Now()` 获取本地时间时，它会返回一个 `time.Time` 和两个时间：

```shell
2021-01-10 17:13:08.852061 +0100 CET m=+0.000338660 
------------------------------------ --------------
Wall time Monotonic time
```

相反，当我们解组 JSON 时，`time.Time` 字段不包含单调时间，只包含墙壁时间。因此，当我们比较两个结构时，结果是错误的，因为时间差是单调的，这也是我们在打印两个结构时注意到差异的原因。

我们如何解决这个问题？有两个主要选项。

当我们使用 `==` 运算符比较两个 `time.Time` 结构字段时，包括单调部分。为了避免这种情况，我们可以改用 `Equal` 方法：

```go
fmt.Println(event1.Time.Equal(event2.Time))
```

```shell
true
```

`Equal` 方法不考虑单调时间；因此，此代码打印为 true。但是，在这种情况下，我们只比较了 `time.Time` 字段，而不是父 `Event` 结构。

第二个选项是保留 `==` 来比较两个结构，但使用 `Truncate` 方法去除单调时间。此方法返回将 `time.Time` 值向下舍入为给定持续时间的倍数的结果。我们可以通过提供零持续时间来使用它，如下所示：

```go
t := time.Now()
event1 := Event{
    Time: t.Truncate(0),
}

b, err := json.Marshal(event1)
if err != nil {
    return err
}

var event2 Event
err = json.Unmarshal(b, &event2)
if err != nil {
    return err
}

fmt.Println(event1 == event2)
```

在此版本中，两个 `time.Time` 字段现在相等。因此，此代码现在将打印为 true。

> **Note** 我们还要注意，每个 `time.Time` 都与一个相关联  `time.Location` 表示时区，例如：
>
> `t := time.Now() // 2021-01-10 17:13:08.852061
+0100 CET`
> 
> 在这里，位置设置为 CET，因为我使用 `time.Now()` 返回我当前的本地时间。我们应该注意到 JSON 封送结果取决于地理位置。如果我们想阻止它，我们应该设置一个特定的地理位置位置，例如：

```go
location, err := time.LoadLocation("America/New_York")
if err != nil {
    return err
}
t := time.Now().In(location) // 2021-05-18 22:47:04.155755 -0500 EST
```

或以 UTC 格式获取当前时间：

```go
t := time.Now().UTC() // 2021-05-18
22:47:04.155755 +0000 UTC
```

总而言之，编组/解组过程并不总是对称的，我们在这种情况下遇到了一个包含 `time.Time` 的结构。我们应该牢记这一原则，这样我们就不会编写错误的测试。

### 10.3.3 any 类型的映射

在解组数据时，我们还可以提供一个映射而不是一个结构。基本原理是当键和值不确定时，传递映射将为我们提供一些灵活性而不是静态结构。然而，为了避免错误的假设和可能的 goroutine panic，有一个特定的规则需要牢记。

让我们编写一个将消息解组到映射中的示例：

```go
b := getMessage()
var m map[string]any
err := json.Unmarshal(b, &m)
if err != nil {
    return err
}
```

如果我们为前面的代码提供以下 JSON：

```json
{
    "id": 32,
    "name": "foo"
}
```

当我们使用通用的 `map[string]any` 时，它会自动解析所有不同的字段：

```shell
map[id:32 name:foo]
```

但是，如果我们使用 `any` 的映射，需要记住一个重要的问题：任何数值，无论它是否包含小数，都将转换为 `float64` 类型。我们可以通过打印 `m[id]` 的类型来观察它：

```shell
fmt.Printf("%T\n", m["id"])
```

```shell
float64
```

我们应该记住它以确保我们不会做出错误的假设，并期望没有小数的数值默认转换为整数。例如，对类型转换做出不正确的假设可能会导致 goroutine panic。

以下部分将深入探讨编写与 SQL 数据库交互的应用程序时最常见的错误。