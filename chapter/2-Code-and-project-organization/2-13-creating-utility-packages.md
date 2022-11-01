## 2.13 创建公共包

本节将讨论一个常见的不良做法：创建共享包，如 `utils`、`common` 或 `base`。我们将了解这种方法的问题，以及如何改进我们的组织。

让我们看看一个受官方Go博客启发的例子。这是关于实现一个集合数据结构（忽略值的map）。在Go中做到这一点的惯用方法是通过带有 `K` 的 `map[K]struct{}` 类型来处理它，`K` 可以是 map 中任何允许的类型，而该值是 `struct{}` 类型。事实上，值类型为 `struct{}` 的映射表明，我们对值本身不感兴趣。让我们在 `util` 包中公开两种方法：

```go
package util

func NewStringSet(...string) map[string]struct{} { ... }
func SortStringSet(map[string]struct{}) []string { ... }
```

client将要像这样使用包:

```go
set := util.NewStringSet("c", "a", "b")
fmt.Println(util.SortStringSet(set))
```

这里的问题是，`util` 毫无意义。我们可以称之为 `common`、`shared` 或 `base`，它仍然是一个毫无意义的名称，无法提供有关软件包提供的任何见解。

我们应该创建一个表达式软件包名称，如 `stringset`，而不是 utility 包，例如：

```go
package stringset

func New(...string) map[string]struct{} { ... }
func Sort(map[string]struct{}) []string { ... }
```

我们删除了 `NewStringSet` 和 `SortStringSet` 的后缀，它们分别变成了 `New` 和 `Sort`。在调用测，它现在看起来是这样的：

```go
set := stringset.New("c", "a", "b")
fmt.Println(stringset.Sort(set))
```

> **Note** 在上一节中，我们讨论了纳米包装的想法。然而，我们提到了在应用程序中创建数十个纳米软件包如何使代码路径更复杂。然而，纳米包装本身的想法并不一定坏。如果一个小型代码组具有高内聚，并且并不真正属于其他地方，那么将其组织成一个特定的软件包是完全可以接受的。没有严格的规则可以适用，通常，挑战在于找到正确的平衡。

我们甚至可以更进一步。我们甚至可以创建特定类型，并以这种方式公开 `Sort` 作为方法，而不是暴露实用程序函数：

```go
package stringset

type Set map[string]struct{}
func New(...string) Set { ... }
func (s Set) Sort() []string { ... }
```

这一变化将使client变得更加简单。`stringset`包只有一个引用：

```go
set := stringset.New("c", "a", "b")
fmt.Println(set.Sort())
```

通过这种小的重构，我们摆脱了一个毫无意义的软件包名称，以公开一个富有表现力的API。

正如Dave Cheney（Go的项目成员）所提到的，找到公用事业软件包来处理公共设施是相当频繁的。例如，如果我们决定有一个 `client` 包和一个 `server` 包，我们应该把常见类型放在哪里？在这种情况下，也许一个解决方案是将客户端、服务器和公共代码合并到一个软件包中。

命名软件包是应用程序设计的关键部分，我们应该对此保持谨慎。根据经验，创建没有有意义名称的共享软件包不是一个好主意；这包括 `utils`、`common` 或者 `base`。此外，让我们记住，以软件包提供的内容而不是包含的内容命名软件包，可以是提高其表现力的有效方法。

在下一节中，我们将继续讨论带有软件包冲突的软件包。