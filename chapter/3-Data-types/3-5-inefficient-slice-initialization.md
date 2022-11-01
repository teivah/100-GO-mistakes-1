## 3.5 低效的切片初始化

在使用 `make` 初始化切片时，我们已经看到我们必须提供长度和可选容量。 在有意义的时候忘记为这两个参数传递一个适当的值是一个普遍的错误。 让我们看看它什么时候被认为是合适的。

让我们看一下下面的例子。 我们将不得不实现一个将 `Foo` 切片映射到 `Bar` 切片的 `convert` 函数。 两个切片将具有相同数量的元素。 这是第一个实现：

```go
func convert(foos []Foo) []Bar {
        bars := make([]Bar, 0)

        for _, foo := range foos {
                bars = append(bars, fooToBar(foo))
        }
        return bars
}
```

首先，我们使用 `make([]Bar, 0)` 初始化 `Bar` 元素的一个空切片。 然后，我们使用 `append` 添加 `Bar` 元素。

起初，`bars` 是空的，因此添加第一个元素将分配一个大小为 1 的后备数组。 然后，每次后备数组已满时，Go 都会通过将其容量翻倍来重新创建另一个数组，如上一节所述。

这种由于当前数组已满而必须创建另一个数组的逻辑将重复多次：当我们添加第 3 个元素、第 5 个、第 9 个等时。假设输入切片有 1000 个元素，该算法需要分配 10 个后备数组 并从一个数组复制到另一个数组，总共超过 1000 个元素。 这也将导致 GC 需要额外的努力来清理所有这些临时后备阵列。

在性能方面，没有充分的理由不向 Go 运行时提供帮助。 有两种不同的选择。

第一种选择是重用相同的代码，但分配给定容量的切片：

```go
func convert(foos []Foo) []Bar {
        n := len(foos)
        bars := make([]Bar, 0, n)

        for _, foo := range foos {
                bars = append(bars, fooToBar(foo))
        }
        return bars
}
```

唯一的变化是创建容量等于 `n` 的条形，即 `foos` 的长度。 在内部，Go 预先分配了一个包含 `n` 个元素的数组。 因此，最多添加 `n` 个元素意味着重用相同的后备数组； 因此，大大减少了分配的数量。

第二个选项是分配给定长度的 `bars`：

```go
func convert(foos []Foo) []Bar {
        n := len(foos)
        bars := make([]Bar, n)

        for i, foo := range foos {
                bars[i] = fooToBar(foo)
        }
        return bars
}
```

当我们用长度初始化切片时，已经分配了 *n* 个元素并将其初始化为 `Foo` 的零值。 因此，要设置元素，我们不必使用 `append` 而是使用 `bar[i]`。

哪个选项是最好的？ 让我们使用三种解决方案和一百万个元素的输入切片运行基准测试：

```shell
BenchmarkConvert_EmptySlice-4                22     49739882 ns/op
BenchmarkConvert_GivenCapacity-4             86     13438544 ns/op
BenchmarkConvert_GivenLength-4               91     12800411 ns/op
```

正如我们所看到的，第一个解决方案在性能方面具有重大影响。 保持分配数组和复制元素使得第一个基准测试比其他两个基准慢 400%。 比较第二种和第三种解决方案，我们可以注意到第三种解决方案的速度大约快 4%，因为我们避免了对 `append` 内置函数的重复调用，与直接赋值相比，它的开销很小。

如果设置容量和使用 `append` 比设置长度和分配给直接索引效率低，那么为什么我们有时会看到它在 Go 项目中使用呢？ 让我们看一下 Pebble 中的一个具体示例，它是 CockroachDB 开发的开源键值存储。

一个名为 collectAllUserKeys 的函数需要遍历一个结构切片来格式化一个特定的字节切片。 结果切片将是输入切片长度的两倍：

```go
func collectAllUserKeys(cmp Compare,
        tombstones []tombstoneWithLevel) [][]byte {
        keys := make([][]byte, 0, len(tombstones)*2)
        for _, t := range tombstones {
                keys = append(keys, t.Start.UserKey)
                keys = append(keys, t.End)
        }
        // ...
}
```

在这里，有意识的选择是使用给定的能力并 `append`。理由是什么？如果我们使用给定的长度而不是容量，代码将如下：

```go
func collectAllUserKeys(cmp Compare,
        tombstones []tombstoneWithLevel) [][]byte {
        keys := make([][]byte, len(tombstones)*2)
        for i, t := range tombstones {
                keys[i*2] = t.Start.UserKey
                keys[i*2+1] = t.End
        }
        // ...
}
```

注意处理切片索引的代码看起来有多复杂。 鉴于此功能对性能不敏感，因此决定使用最容易阅读的选项。

> **Note** 如果无法准确知道切片的未来长度怎么办？ 例如，如果输出切片的长度取决于一个条件：

```go
func convert(foos []Foo) []Bar {
// bars initialization

        for _, foo := range foos {
                if something(foo) {
                        continue
                }
                // Add a bar element
        }
        return bars
}
```

> 在该示例中，`Foo` 元素被转换为 `Bar` 并仅在特定条件下（ `if something(foo)` ）添加到切片中。 我们应该将 `bars` 初始化为空切片还是具有给定长度或容量？ 那里没有严格的规定。 换CPU好还是内存好，这是一个传统的软件问题。 也许 `if something(foo)` 在 99% 的情况下为真，那么值得用长度或容量来初始化 `bars` 吗？ 这取决于我们的用例。

将一种切片类型转换为另一种切片类型是 Go 开发人员的常见操作。 如我们所见，如果未来切片的长度已知，则没有充分的理由首先分配一个空切片。 选项是分配给定容量或给定长度的切片。 在这两种解决方案之间，我们已经看到最后一种解决方案往往会稍微快一些。 然而，在某些情况下，使用给定的容量和 `append` 可能更容易实现和阅读。

下一节将讨论零切片和空切片之间的区别，以及为什么这对Go开发人员很重要。