## 2.8 `any` say nothing

在 Go 中，指定零方法的接口类型称为空接口：`interface{}`。从 Go 1.18开始，预声明的类型 `any` 已成为空接口的别名。因此，所有 `interface{}` 事件都可以替换为 `any`。然而，在许多情况下，`any` 都可以被视为过度概括。让我们首先关注我们核心概念，然后我们将讨论潜在的问题。

一个 `any` 类型可以容纳任何类型的值：

```go
func main() {
    var i any

    i = 42
    i = "foo"
    i = struct {
        s string
    }{
        s: "bar",
    }
    i = f
    
    _ = i
}

func f() {}
```

在为 `any` 类型分配值时，我们会丢失所有类型信息，这些信息需要类型断言才能从 `i` 变量中获得任何有用的东西。

现在，让我们看到一个具体的例子，其中使用 `any` 不准确。我们将实现一个 `Store` 结构和两种方法的骨架：`Get` 和 `Set` 。这些方法将用于存储不同的结构类型：`Customer` 和 `Contract`：

```go
package store

type Customer struct{
    // Some fields
}
type Contract struct{
    // Some fields
}

type Store struct{}

func (s *Store) Get(id string) (any, error) {
    // ...
}

func (s *Store) Set(id string, v any) error {
	// ...
}
```

虽然 `Store` 编译没有问题，但我们应该花一分钟来考虑方法定义。当我们接受并返回 `any` 参数时，这些方法缺乏表达性。如果未来的开发人员必须使用 `Store` 结构，他们可能需要深入研究文档或阅读代码，以了解如何使用这些方法。因此，接受或返回 `any` 类型并不能传达有意义的信息。

此外，由于编译时没有保障措施，因此没有任何因素阻止调用者使用任何数据类型调用这些方法，例如一个 `int` 类型的值：

```go
s := store.Store{}
s.Set("foo", 42)
```

通过使用 `any`，我们失去了 Go 作为静态类型语言的一些好处。相反，我们应该避免 `any` 类型，并尽可能明确我们的定义。关于我们的例子，这可能意味着复制每种类型的获取和设置方法：

```go
func (s *Store) GetContract(id string) (Contract, error) {
    // ...
}

func (s *Store) SetContract(id string, contract Contract) error {
    // ...
}
func (s *Store) GetCustomer(id string) (Customer, error) {
    // ...
}

func (s *Store) SetCustomer(id string, customer Customer) error {
    // ...
}
```

在这个版本中，这些方法富有表现力，降低了不理解的风险。拥有更多方法不一定是一个问题，因为 client 也可以使用接口创建自己的抽象。例如，如果 client 只对 `Contract` 方法感兴趣：

```go
type ContractStorer interface {
    GetContract(id string) (store.Contract, error)
    SetContract(id string, contract store.Contract) error
}
```

那么，当有 `any` 案例有帮助时，有哪些情况呢？让我们看看标准库，看看函数或方法接受 `any` 参数的两个示例。

第一个例子是在 `encoding/json` 包中。由于我们可以调用任何类型，`Marshal` 函数接受 `any` 参数：

```go
func Marshal(v any) ([]byte, error) {
    // ...
}
```

另外一个例子是在 `database/sql` 包中，如果查询是参数化的(例如：`SELECT * FROM FOO WHERE id = ?`)，则参数可以是任何类型的。因此，它还使用 `any` 参数。

```go
func (c *Conn) QueryContext(ctx context.Context, query string,
    args ...any) (*Rows, error) {
    // ...
}
```

总之，如果真的需要接受或退回任何可能的类型，`any` 都可以提供帮助。例如，当涉及到到变量重组或格式化时。一般来说，我们应该避免不惜一切代价过度概括我们编写的代码。也许，如果能改善代码表达性等其他方面，一点重复的代码偶尔会更好。

接下来的章节将讨论另一种类型的抽象：泛型。