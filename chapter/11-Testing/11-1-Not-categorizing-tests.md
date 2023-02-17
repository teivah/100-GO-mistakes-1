# 11 测试

> **本章概要**
> * 深入研究有用的 Go 选项以对测试进行分类或使其更健壮
> * 通过了解如何防止使用睡眠调用并了解如何使用时间 API 测试函数，使 Go 测试具有确定性
> * 深入研究实用程序包，例如 `httptest` 和 `iotest`
> * 讨论导致错误假设的常见基准错误
> * 提出一组改进测试过程的技巧

测试是项目生命周期的一个重要方面。它有无数的好处，例如建立对应用程序的信心、充当代码文档或使重构更容易。与其他一些语言相比，Go 具有强大的原语来编写测试。在本节中，我们将深入研究导致测试过程脆弱、效率降低和准确性降低的常见错误。

## 11.1 测试也需要分类

测试金字塔是一种将测试分为不同类别的模型。单元测试占据了金字塔的底部。大多数测试应该是单元测试；因此，编写成本低、执行速度快且具有高度确定性。通常，我们在金字塔中越向上，编写的测试就越复杂，运行起来越慢，并且越难保证它们的确定性：

![](https://img.exciting.net.cn/73.png)

一种常见的技术是明确说明要运行哪种测试。实际上，根据项目生命周期阶段，我们可能希望仅运行单元测试或运行项目中的所有测试。不对测试进行分类意味着可能会浪费时间和精力，并且会失去测试范围的准确性。本节将深入探讨 Go 中对测试进行分类的三种主要方法。

### 11.1.1 构建标签

对测试进行分类的最常用技术是使用构建标签。构建标签是 Go 文件开头的特殊注释，后跟一个空行。

```go
//go:build foo

package bar
```

此文件包含 `foo` 标记。请注意，在一个包中，我们可能有多个具有不同构建标签的文件。

> **Note** 在 Go 1.17 中，原始语法 `// +build foo` 被 `//go:build foo` 取代。截止到当前版本（Go 1.18），`gofmt` 可以同步使用这两种形式来辅助程序迁移到 Go 1.18。

构建标签用于两个主要用例。首先，作为构建应用程序的条件选项；例如，如果我们希望仅在启用 cgo 时才包含源文件（cgo 是一种让 Go 包调用 C 代码的方法），我们可以添加 `//go:build cgo` 构建标签。第二个用例是如果我们想将测试归类为集成测试，我们还可以添加特定的构建标志，例如 `integration`：

```go
//go:build integration

package db

import (
    "testing"
)

func TestInsert(t *testing.T) {
    // ...
}
```

在这里，我们添加了集成构建标签来分类该文件包含集成测试。使用构建标签的好处是我们可以选择执行哪种测试。例如，假设一个包包含两个测试文件：

* 我们刚刚创建的 `db_test.go`
* 加上另一个不包含任何构建标签的 `contract_test.go`

如果我们在这个包中运行 `go test` 而没有任何选项，它将只运行没有构建标签的测试文件；因此 `contract_test.go`：

```shell
$ go test -v .
=== RUN   TestContract
--- PASS: TestContract (0.01s)
PASS
```

但是，如果我们现在提供 `integration` 标签，它还将包含 `db_test.go`：

```shell
$ go test --tags=integration -v .
=== RUN   TestInsert
--- PASS: TestInsert (0.01s)
=== RUN   TestContract
--- PASS: TestContract (2.89s)
PASS
```

因此，使用特定标签运行测试包括不带标签的文件和匹配此标签的文件。现在，如果我们只想运行集成测试怎么办？一种可能的方法是在单元测试文件上添加否定标记。例如，使用 `!integration` 意味着仅在 *未* 启用 `integration` 标志时才包含测试文件：

```go
//go:build !integration

package db

import (
    "testing"
)

func TestContract(t *testing.T) {
    // ...
}
```

因此，运行 `go test`：

* 使用 `integration` 标志将仅运行集成测试
* 没有 `integration` 标志将只运行单元测试

现在让我们讨论另一个在单个测试级别，而不是文件级别上工作的选项。

### 11.1.2 环境变量

正如 Go 社区的成员 Peter Bourgon 所提到的，构建标签有一个主要缺点：缺少测试已被忽略的信号。在第一个示例中，当我们执行没有构建标志的 `go test` 时，它只显示了已执行的测试：

```shell
$ go test -v .
=== RUN   TestUnit
--- PASS: TestUnit (0.01s)
PASS
ok      db  0.319s
```

如果我们对标签的处理方式不够小心，我们可能会忘记现有的测试。出于这个原因，一些项目倾向于使用环境变量检查测试类别的方法。

例如，我们可以通过检查特定环境变量并可能跳过测试来实现 `TestInsert` 集成测试：

```go
func TestInsert(t *testing.T) {
    if os.Getenv("INTEGRATION") != "true" {
        t.Skip("skipping integration test")
    }

    // ...
}
```

如果 `INTEGRATION` 环境变量未设置为 true，则会跳过测试并显示一条消息：

```shell
$ go test -v .
=== RUN   TestInsert
    db_integration_test.go:12: skipping integration test
--- SKIP: TestInsert (0.00s)
=== RUN   TestUnit
--- PASS: TestUnit (0.00s)
PASS
ok      db  0.319s
```

使用这种方法的一个好处是明确跳过的测试及其原因。这种技术可能不如构建标签广泛使用，但值得了解，因为它具有一些优势，正如我们所讨论的。

现在让我们深入研究另一种对测试进行分类的方法：短模式。

### 11.1.3 短模式

另一种对测试进行分类的方法与它们的速度有关。事实上，我们可能不得不将短期测试与长期运行的测试区分开来。

作为一个例子，我们可以有一组单元测试，但只有一个是出了名的慢。因此，我们希望以特定方式对其进行分类，这样我们就不必每次都运行它（特别是如果触发器是在保存文件之后）。为此，短模式允许我们进行区分：

```go
func TestLongRunning(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping long-running test")
    }
    // ...
}
```

使用 `testing.Short`，我们可以检索在运行测试时是否启用了短模式。然后，我们使用 `Skip` 跳过测试。要使用短模式运行测试，我们必须通过 `‑short`：

```shell
% go test -short -v .
=== RUN TestLongRunning
foo_test.go:9: skipping long-running test
--- SKIP: TestLongRunning (0.00s)
PASS
ok foo 0.174s
```

`TestLongRunning` 被排除并从执行的测试中显式跳过。请注意，与构建标签相比，此选项适用于每个测试，而不是每个文件。

总之，对测试进行分类是成功测试策略的最佳实践。在本节中，我们看到了三种对测试进行分类的方法：

* 在测试文件级别使用构建标签
* 使用环境变量来标记特定的测试
* 基于使用短模式的测试速度

我们也可以结合这两种方法。例如，如果我们的项目包含长时间运行的单元测试，则使用构建标签或环境变量对测试（例如，单元测试或集成测试）和短模式进行分类。

在下一节中，我们将讨论为什么启用 `rase` 标志很重要。