## 3.13 错误的比较值

比较值是软件开发中的常见操作。无论是在比较两个对象的函数中，还是在将值与期望值进行比较的测试中，实现比较都非常频繁。我们的第一直觉可能是在任何地方都使用 `==` 运算符。然而，正如我们将在本节中看到的那样，情况并非总是如此。那么什么时候使用比较合适 `==`，还有什么选择？

让我们从一个具体的例子开始。我们将创建一个基本的 `custormer` 结构并使用 `==` 来比较两个实例。在您 看来，这段代码的输出应该是什么？

```go
type customer struct {
        id string
}

func main() {
        cust1 := customer{id: "x"}
        cust2 := customer{id: "x"}
        fmt.Println(cust1 == cust2)
}
```

在 Go 中比较这两个 `custormer` 结构是一个有效的操作，它会打印 true。现在，如果我们对 `custormer` 结构添加切片字段：

```go
type customer struct {
        id         string
        operations []float64
}

func main() {
        cust1 := customer{id: "x", operations: []float64{1.}}
        cust2 := customer{id: "x", operations: []float64{1.}}
        fmt.Println(cust1 == cust2)
}
```

我们可能希望这段代码也打印出 true。但是，它甚至没有编译：

```go
invalid operation:
    cust1 == cust2 (struct containing []float64 cannot be compared)
```

问题与 `==` 和 `!=` 运算符的工作方式有关。此运算符不适用于切片或映射。因此，由于 `custormer` 结构包含一个切片，它不会编译。

了解如何使用 `==` 和 `!=` 进行有效比较至关重要。我们可以在可比较的操作数上使用它们：

* 布尔型：比较两个布尔值是否相等。
* 数字（int、float和complex类型）：比较两个数值是否相等。
* 字符串：比较两个字符串是否相等。
* 通道：比较两个通道是由同一次调用 `make` 创建的，还是两者都为nil。
* 接口：比较两个接口是否具有相同的动态类型和相等的动态值，或者两者的值是否为零。
* 指针：：比较两个指针是指向内存中的相同值还是两者都为nil。
* 结构和数组：如果他们由可比较的类型组成。

> **Note** 我们也可以使`<=`,`>=`,`<` 和 `>` 用运算符用于比较数值类型和字符串以比较它们的词法顺序。

在最后一个示例中，我们的代码无法编译，因为结构是由不可比较的类型（切片）组成的。

我们还应该知道将 `==` 和 `!=` 用于 `any` 类型的可能问题。例如，允许比较分配给 `any` 类型的两个整数：

```go
var a any = 3
var b any = 3
fmt.Println(a == b)
```

代码打印：

`true`

但是，如果我们初始化两个 `custormer` 类型（包含切片字段的最新版本）并将值分配给 `any` 类型怎么办？

```go
var cust1 any = customer{id: "x", operations: []float64{1.}}
var cust2 any = customer{id: "x", operations: []float64{1.}}
fmt.Println(cust1 == cust2)
```

此代码可以编译。然而，由于 `custormer` 结构包含切片字段，因此无法比较这两种类型，因此会在运行时导致错误，即panic：

```go
panic: runtime error: comparing uncomparable type main.customer
```

考虑到这些行为，如果我们必须比较两个切片、两个映射或两个包含不可比较类型的结构，有哪些选择？如果我们坚持使用标准库，另一种选择是用 `reflect` 包使用运行时反射。
反射是元编程的一种形式，它指的是应用程序自省和修改其结构和行为的能力。

例如，在 Go 中，我们可以使用 `reflect.DeepEqual`。此函数通过递归遍历两个值来报告两个元素是否深度相等。它接受的元素是基本类型加上数组、结构、切片、映射、指针、接口和函数。

> **Note** `reflect.DeepEqual` 具有特定的行为，具体取决于我们提供的类型。在使 用它之前，让我们仔细阅读文档。

让我们重新运行第一个示例，使用 `reflect.DeepEqual`：

```go
cust1 := customer{id: "x", operations: []float64{1.}}
cust2 := customer{id: "x", operations: []float64{1.}}
fmt.Println(reflect.DeepEqual(cust1, cust2))
```

在这里，即使 `custormer` 结构包含不可比较的类型（切片），它也会按预期运行并打印为 true。

不过使用 `reflect.DeepEqual` 时有两点需要注意。第一个是它区分了空和 nil 集合，如对 nil 与空切片感到困惑中所述。这是一个问题吗？不必要;这取决于我们的用例。例如，如果我们想比较两个解组操作的结果（例如，从 JSON 到 Go 结构），我们可能希望提高这种差异。然而，有效地使用这种行为是值得的 `reflect.DeepEqual`。

在大多数语言中，另一个问题是相当标准的。由于此函数使用反射，它将在运行时自省这些值以发现它们是如何形成的，因此它会降低性能。平均而言，在本地使用不同大小的结构进 行一些基准测试，这可能是一个 `reflect.DeepEqual` 比 `==` 慢大约100倍。有理由支持在测试而不是运行时使用它。

如果性能是一个关键因素，另一种选择可能是实现我们自己的比较方法。这是一个比较两个 `custormer`  结构并返回布尔值的示例：

```go
func (a customer) equal(b customer) bool {
        if a.id != b.id {
                return false
        }
        if len(a.operations) != len(b.operations) {
                return false
        }
        for i := 0; i < len(a.operations); i++ {
                if a.operations[i] != b.operations[i] {
                        return false
                }
        }
        return true
}
```

在这段代码中，我们使用自定义检查 `custormer`  结构的不同字段来构建我们的比较方法。在由 100 个元素组成的切片上运行本地基准测试表明，我们自定义的 `equal` 方法比 `reflect.DeepEqual` 快了大约96倍。

一般来说，我们应该记住 `==` 运算符非常有限。 例如，它不适用于切片和地图。 在大多数情况下，使用 `reflect.DeepEqual` 是一种解决方案，但主要问题是性能损失。 在单元测试的上下文中，其他一些选项是可能的，例如使用带有 `go-cmp` [9] 或 `testify` [10] 的外部库。 但是，如果性能在运行时至关重要，那么实现我们的自定义方法可能是最好的解决方案。 另外一点：我们还应该记住标准库有一些现有的比较方法。 例如，我们可以使用优化的 `bytes.Compare` 函数来比较两个字节切片。 让我们确保我们不会重新发明轮子。