## 3.6 令人困惑的切片，nil vs 空切片

Go 开发人员经常混合使用 nil 和空切片。根据用例，我们可能希望使用其中一种。同时，一些类库对两者进行了区分。因此，要精通切片，我们需要确保不要混淆这些概念。

在看一个例子之前，让我们讨论一下定义：

* 如果切片长度为零，则切片为空
* 如果切片等于 `nil`，则切片为 nil

让我们看看初始化切片的不同方法。你能猜出这段代码的输出是什么吗？每次，我们都会打印这个切片是空还是 nil：

```go
func main() {
        var s []string
        log(1, s)

        s = []string(nil)
        log(2, s)

        s = []string{}
        log(3, s)

        s = make([]string, 0)
        log(4, s)
}

func log(i int, s []string) {
        fmt.Printf("%d: empty=%t\tnil=%t\n", i, len(s) == 0, s == nil)
}
```

这个例子打印如下：

```shell
1: empty=true   nil=true
2: empty=true   nil=true
3: empty=true   nil=false
4: empty=true   nil=false
```

所有切片都是空的，这意味着长度为零。因此，一个 nil 切片也是一个空切片。但是，只有前两个是 nil 切片。那么，如果我们有多种方法来初始化一个切片，我们应该选择哪个选项呢？

有两点需要注意：

* 首先，nil 和空切片之间的主要区别之一是关于分配。实际上，初始化一个 nil 切片不需要任何分配，而空切片则不是这样。
* 其次，无论切片是否为 nil，调用 append 内置函数都可以：

```go
var s1 []string
fmt.Println(append(s1, "foo")) // [foo]
```

因此，例如，如果一个函数返回一个切片，出于防御原因，我们不应该像在其他语言中那样返回一个非零集合。由于 nil 切片不需要任何分配，我们应该倾向于返回一个 nil 切片。让我们看看这个返回字符串切片的函数：

```go
func f() []string {
        var s []string
        if foo() {
                s = append(s, "foo")
        }
        if bar() {
                s = append(s, "bar")
        }
        return s
}
```

如果 `foo` 和 `bar` 都为 false，我们将返回一个空切片。在这里，为了防止无缘无故分配空切片，我们应该支持选项一（`var s []string`）。可以使用长度为零的第四方案 (`make([]string, 0)`)，但与第一方案相比，这不会带来任何价值，并且它需要分配，而这里不是这种情况。

但是，如果我们必须生成一个已知长度的切片，我们应该使用第四方案 (`s := make([]string, length[, capacity])`)：

```go
func intsToStrings(ints []int) []string {
        s := make([]string, len(ints))
        for i, v := range ints {
                s[i] = strconv.Itoa(v)
        }
        return s
}
```

实际上，正如在低效率切片初始化中所讨论的，我们应该在这种情况下设置长度（或容量）以避免额外的分配和复制。

现在，还有两个不同的选择：

* 选项二： `s := []string(nil)`
* 选项三：`s := []string{}`

第一个选项不是使用最广泛的。然而，它对语法糖很有帮助，因为我们可以在一行中传递一个 nil 切片。例如，使用 `append`：

```go
s := append([]int(nil), 42)
```

如果我们使用选项一（`var s []string`），则需要两行代码。这可能不是有史以来最重要的可读性优化； 然而，它仍然值得知道。

> **Note** 我们将在未正确制作切片副本中看到附加到零切片的一个基本原理。

现在，最新的选项：选项三，`s := []string{}`。建议使用此表单创建一个包含初始元素的切片。例如：

```go
s := []string{"foo", "bar", "baz"}
```

但是，如果我们不需要使用初始元素创建它，则不应使用此选项。实际上，它带来了与选项一 (`var s []string`) 相同的好处，只是切片不是 nil； 因此，它需要分配。因此，应避免没有初始元素的选项三。

> **Note**  一些 linter 可以在没有初始值的情况下捕获选项三，并建议在选项一中更改它。但是，请注意，它还将语义从非 nil 更改为 nil 切片。

我们还应该注意，一些库区分 nil 和空切片。例如， `encoding/json` 包就是这种情况。让我们看一下以下示例，我们将在其中编组两个结构：一个包含 nil 切片，而第二个包含非 nil 空切片：

```go
var s1 []float32
        customer1 := customer{
        ID:         "foo",
        Operations: s1,
}
b, _ := json.Marshal(customer1)
fmt.Println(string(b))

s2 := make([]float32, 0)
        customer2 := customer{
        ID:         "bar",
        Operations: s2,
}
b, _ = json.Marshal(customer2)
fmt.Println(string(b))
```

通过此示例，我们可以注意到这两个结构的重组结果不同：

```json
{"ID":"foo","Operations":null}
{"ID":"bar","Operations":[]}
```

`nil` 切片被编组为空元素，而非 nil 空切片被编组为空数组。如果我们在严格区分 null 和 [] 的 JSON 客户端的上下文中工作，则必须牢记这一区别。

`encoding/json` 并不是标准库中唯一可以区分的包。例如，如果我们比较一个 nil 和一个非 nil 空切片，则 `reflect.DeepEqual` 返回 false，这在单元测试的上下文中是要记住的。无论如何，在使用标准库或外部库时，我们应该确保使用一个或另一个版本不会导致意外结果。

总而言之，在 Go 中，零切片和空切片是有区别的。nil 切片等于 `nil`，而空切片的长度为零。nil 切片是空的，但空切片不一定是 nil。同时，零切片不需要任何分配。我们已经在本节中看到了如何根据上下文初始化切片：

* `var s []string` 如果我们不确定最终的长度，切片可以是空。
* `[]string(nil)` 作为句法糖来创造一个零和空的切片。
* `make([]string, length[, capacity])` 如果未来的长度是确定的。

如果我们在没有元素的情况下初始化切片，则应避免使用最后一个选项 `[]string{}`。

最后但同样重要的是，让我们确保检查我们使用的库是否区分了 nil 和空切片以防止意外行为。

在下一节中，我们将继续这个讨论，看看调用函数后检查空切片的最佳方法。