## 2.9 令人困惑的泛型

截至 Go 1.18，泛型已添加到该语言中。简而言之，它允许编写具有类型代码，这些类型可以稍后指定，并在需要时实例化。然而，很容易对何时使用泛型和何时不使用泛型感到困惑。在本节中，我们将在 Go 中描述泛型的概念，然后深入研究常见用途和滥用。

### 2.9.1 概念

考虑以下从 `map[string]int` 类型中提取所有键的函数：

```go
func getKeys(m map[string]int) []string {
    var keys []string
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}
```

如果我们想将类似的功能用于其他 map 类型，例如 `map[int]string`，该怎么办？在泛型之前，Go 开发人员有几个选项：使用代码生成、反射或复制代码。

例如，我们可以编写两个函数，每个映射类型一个，甚至尝试扩展 `getKeys` 以接受不同的 map 类型：

```go
func getKeys(m any) ([]any, error) {
    switch t := m.(type) {
    default:
        return nil, fmt.Errorf("unknown type: %T", t)
    case map[string]int:
        var keys []any
        for k := range t {
            keys = append(keys, k)
        }
        return keys, nil
    case map[int]string:
        // ...
    }
}
```

我们可以开始注意到几个问题：

* 首先，它增加了模版化的代码。事实上，每当我们想添加一种类型时，都需要复制 `range` 循环。
* 与此同时，函数现在接受 `any` 类型，这意味着我们正在失去 Go 作为类型化语言的一些优势。实际上，检查类型是否受支持是在运行时而不是编译时进行的。因此，如果提供的类型未知，我们还需要返回错误。
* 最后，由于 key 类型可以是 `int` 或 `string`，我们有义务返回 `any` 类型的切片来分解 key 类型。这种方法增加了调用侧的工作量，因为 client 可能还必须对 key 进行类型检查或进行额外的转换。

多亏了泛型，我们现在可以使用类型参数重构此代码。

类型参数是我们可以与函数和类型一起使用的泛型类型。例如，以下函数接受类型参数：

```go
func foo[T any](t T) {
    // ...
}
```

调用 `foo` 时，我们将传递任何类型的类型参数。提供类型参数称为实例化，工作在编译时完成，这使类型安全作为核心语言功能的一部分，并避免了运行时开销。

让我们回到 `getKeys` 函数，并使用类型参数编写一个接受任何类型 map 的通用版本：

```go
func getKeys[K comparable, V any](m map[K]V) []K {
    var keys []K
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}
```

为了处理 map，我们定义了两种类型参数。首先，值可以是任何类型：`V any`。然而，在 Go 中，map 键不能是任何类型的。例如，我们不能使用切片：

```go
var m map[[]byte]int
```

此代码导致编译错误：`invalid map key type []byte`。因此，我们有义务限制类型参数，以便密钥类型满足特定要求，而不是接受任何密钥类型。在这里，具有可比性（我们可以使用 `==` 或 `!=` ）。因此，我们将 `K` 定义为 `comparable`，而不是 `any`。

限制类型参数以匹配特定需求称为约束。约束是一种接口类型，可以包含：

* 一组行为（方法）
* 但也是任意类型

让我们看看后者的具体例子。想象一下，我们不想接受任何 `comparable` 类型的 map key 类型。例如，我们希望将其限制为 `int` 或 `string` 类型。我们可以这样定义自定义约束：

```go
type customConstraint interface {
    ~int | ~string
}

func getKeys[K customConstraint, V any](m map[K]V) []K {
    // Same implementation
}
```

首先，我们定义了一个 `customConstraint` 接口，以使用联合运算符 `|` 将类型限制为 `int` 或 `string` （稍后我们将讨论 `~` 的使用）。然后，`K` 现在是一个 `customConstraint`，而不是像以前那样的 `comparable`。

现在，`getKeys` 的定义强制要求我们可以使用任何值类型的映射调用它，但密钥类型必须是 `int` 或 `string`。例如，在调用侧：

```go
m = map[string]int{
    "one":   1,
    "two":   2,
    "three": 3,
}
keys := getKeys(m)
```

请注意，Go 可以推断 `getKeys` 是使用 `string` 类型参数调用的。上面的调用等同于此：

```go
keys := getKeys[string](m)
```

> **Note** 使用此约束会将类型参数限制为像这样的自定义类型：

```go
type customInt int

func (i customInt) String() string {
    return strconv.Itoa(int(i))
}
```

由于 `customInt` 是一个 `int` 并实现了 `String() string` 方法，`customInt` 类型满足定义的约束。

但是，如果我们将约束更改为包含 `int` 而不是 `~int`，使用 `customInt` 将导致编译错误，因为 `int` 类型没有实现 `String() string`。

使用 `~int` 或 `int` 的约束有什么区别？使用 `int` 将其限制为该类型，而 `~int` 限制所有底层类型为 `int` 的类型。

为了说明这一点，让我们想象一个约束，即我们希望将类型限制为实现 `String() string` 方法的任何 `int` 类型：

```go
type customConstraint interface {
    ~int
    String() string
}
```

我们还请注意，`constraints` 包包含一组常见的约束，例如 `Signed`，其中包括所有有符号整数类型。在创建新软件包之前，请确保此软件包中不存在约束。

到目前为止，我们已经讨论了将泛型用于函数的示例。然而，我们也可以使用具有数据结构的泛型。

例如，我们将创建一个包含任何类型值的链接列表。同时，我们将编写一个 `Add` 方法来附加节点：

```go
type Node[T any] struct {
    Val  T
    next *Node[T]
}

func (n *Node[T]) Add(next *Node[T]) {
    n.next = next
}
```

我们使用类型参数来定义 `T`，并在 `Node` 中使用两个字段。关于该方法，实例化了接收器。事实上，由于 `Node` 是泛型的，它还必须遵循定义的类型参数。

关于类型参数，最后需要注意的是，它们不能与方法参数一起使用，只能与函数参数或方法接收器一起使用。例如，以下方法不会编译：

```go
type Foo struct {}

func (Foo) bar[T any](t T) {}
```

`./main.go:29:15: methods cannot have type parameters`

如果我们想将泛型与方法一起使用，则接收器需要成为类型参数。

现在，让我们深入研究我们应该和不应该使用泛型的具体案例。

## 2.9.2 常见用途和误用

那么，泛型什么时候有用呢？让我们讨论一下推荐泛型的几种常见用途：

* 数据结构。例如，如果我们实现二叉树、链接列表或堆，我们可以使用泛型来分解元素类型。

* 处理任何类型的切片、map 和通道的函数。例如，合并两个通道的函数将适用于任何通道类型。因此，我们可以使用类型参数来分解通道类型：

```go
func merge[T any](ch1, ch2 <-chan T) <-chan T {
    // ...
}
```

* 分解行为而不是类型。例如，`sort` 包包含一个带有三种方法的 `sort.Interface` 接口：

```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

此接口由 `sort.Ints` 或 `sort.Float64s` 等不同功能使用。使用类型参数，我们可以考虑排序行为。例如，通过定义一个持有切片和比较函数的结构：

```go
type SliceFn[T any] struct {
    S       []T
    Compare func(T, T) bool 
}

func (s SliceFn[T]) Len() int           { return len(s.S) }
func (s SliceFn[T]) Less(i, j int) bool { return s.Compare(s.S[i], s.S[j]) }
func (s SliceFn[T]) Swap(i, j int)      { s.S[i], s.S[j] = s.S[j], s.S[i] }
```

然后，当 `SliceFn` 结构实现 `sort.Interface` 时，我们可以使用 `sort.Sort（sort.Interface）` 函数对提供的切片进行排序：

```go
s := SliceFn[int]{
    S: []int{3, 2, 1},
    Compare: func(a, b int) bool {
            return a < b
    },
}
sort.Sort(s)
fmt.Println(s.S)
```

`[1 2 3]`

在本例中，分解行为可以让我们避免为每种类型创建一个函数。

相反，什么时候建议不要使用泛型？

* 当只是调用类型参数的方法时。例如，考虑一个接收 `io.Writer` 并调用 `Write` 方法的函数：

```go
func foo[T io.Writer](w T) {
    b := getBytes()
    _, _ = w.Write(b)
}
```

在这种情况下，使用泛型不会带来任何价值，我们应该直接将 `w` 参数设置为 `io.Writer` 。

* 当它使我们的代码更加复杂时。通用从来都不是强制性的，作为 Go 开发人员，我们已经能够在没有它们的情况下生活了十多年。如果编写通用函数或结构，我们发现它没有使我们的代码更清晰，我们可能应该重新考虑我们对这个特定用例的决定。

虽然泛型在特定条件下非常有帮助，但我们应该谨慎对待何时使用它们，而不是使用它们。一般来说，当我们想回答何时不使用泛型时，我们可以找到与何时不使用接口的相似之处。事实上，泛型引入了一种抽象形式，我们必须记住，不必要的抽象引入了复杂性。同样，让我们不要用不必要的抽象来污染我们的代码，现在让我们专注于解决具体问题。这意味着我们不应该过早使用类型参数。让我们等到我们即将编写样板代码时再考虑使用泛型。

在接下来的章节中，我们将讨论使用类型嵌入时可能出现的问题。