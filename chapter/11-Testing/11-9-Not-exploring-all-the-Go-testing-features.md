## 11.9 没有探索所有的 Go 测试特性

在编写测试时，开发人员应该了解 Go 的特定测试功能和选项。否则，测试过程可能不太准确，甚至效率较低。本章的最后一节将深入探讨可以让我们在编写 Go 测试时更加自如的不同部分。

### 11.9.1 代码覆盖率

在开发过程中，可以很方便地直观地查看代码的哪一部分被测试覆盖。我们可以使用 `coverprofile` 标志访问这些信息：

```shell
$ go test -coverprofile=coverage.out ./...
```

此命令创建一个 `coverage.out` 文件，然后我们可以使用 `go tool cover` 打开该文件：

```shell
$ go tool cover -html=coverage.out
```

此命令将打开 Web 浏览器并显示每行代码的覆盖率。

要添加的一件事，默认情况下，代码覆盖率仅分析当前正在测试的包。例如，如果我们有以下结构：

```shell
/myapp
  |_ foo
    |_ foo.go
    |_ foo_test.go
  |_ bar
    |_ bar.go
    |_ bar_test.go
```

如果 `foo.go` 的某些部分仅在 `bar_test.go` 中进行测试，默认情况下，它不会显示在覆盖率报告中。 要包含它，我们必须在 `myapp` 文件夹中并使用 `coverpkg` 标志：

```shell
go test -coverpkg=./... -coverprofile=coverage.out ./...
```

让我们记住这个特性以查看当前的代码覆盖率并决定哪些部分值得更多测试。

> **Note** 在追逐代码覆盖率方面，我们应该保持谨慎。拥有 100% 的测试覆盖率并不意味着应用程序没有错误。正确推理我们的测试涵盖的内容比任何静态阈值都更重要。

### 11.9.2 从不同的包进行测试

在编写单元测试时，一种方法是关注行为而不是内部。事实上，假设我们向客户公开了一个 API。我们可能希望我们的测试专注于从外部可见的内容，而不是实现细节。这样，如果实现发生变化（例如，将一个函数重构为两个），测试将保持不变。此外，它们可以更容易理解，因为它们展示了我们的 API 是如何使用的。如果我们想强制执行这种做法，我们可以使用不同的包来实现。

在 Go 中，一个文件夹中的所有文件都应该属于同一个包，只有一个例外：一个 测试文件可以属于一个 `_test` 包。

例如，如果以下源文件属于 `counter` 包:

```go
package counter

import "sync/atomic

var count uint64

func Inc() uint64 {
	atomic.AddUint64(&count, 1)
	return count
}
```

测试文件可以存在于同一个包中，也可以访问内部变量，例如 `count` 变量。或者，它可以存在于 `counter_test` 包中，如下所示：

```go
package counter_test

import (
    "testing"

    "myapp/counter"
)

func TestCount(t *testing.T) {
    if counter.Inc() != 1 {
        t.Errorf("expected 1")
    }
}
```

在这种情况下，测试是在外部包中实现的，并且无法访问内部变量，例如 `count` 变量。

使用这种做法，我们可以保证测试不会使用任何未导出的元素；因此，它将专注于 测试暴露的行为。

### 11.9.3 通用函数

在编写测试时，我们可以以不同于我们的生产代码的方式处理错误。

例如，假设我们要测试一个将 `Customer` 结构作为参数的函数。由于 `Customer` 的创建将被重用，我们将决定为了测试创建一个特定的 `createCustomer` 函数。此函数将返回一个可能的错误以及一个 `Customer` ：

```go
func TestCustomer(t *testing.T) {
    customer, err := createCustomer("foo")
    if err != nil {
        t.Fatal(err)
    }
    // ...
}

func createCustomer(someArg string) (Customer, error) {
    // Create customer
    if err != nil {
        return Customer{}, err
    }
    return customer, nil
}
```

我们使用 `createCustomer` 实用函数创建客户，然后执行其余测试。但是，在测试功能的上下文中，我们可以简化错误管理。我们可以通过将 `*testing.T` 变量传递给实用程序函数来做到这一点：

如果 `createCustomer` 无法创建 `Customer`，它不会返回错误，而是直接使测试失败。这样，它使 `TestCustomer` 更小，更易于编写和阅读。

让我们记住这种关于错误管理和测试的实践，以改进我们的测试。

### 11.9.4 安装与卸载

在某些情况下，我们可能需要准备一个测试环境。例如，在集成测试中，启动特 定的 Docker 容器然后停止它。关于安装与卸载功能，我们可能希望每个测试或每个包 都这样做。幸运的是，在 Go 中，两者都是可能的。

为了每次测试都做，我们可以调用 setup 函数作为 preaction 并使用 defer 调用 teardown 函数：

```go
func TestMySQLIntegration(t *testing.T) {
    setupMySQL()
    defer teardownMySQL()
    // ...
}
```

我们应该注意到，也可以注册一个要在测试结束时执行的函数。例如，假设 `TestMySQLIntegration` 需要调用 `createConnection` 来创建数据库连接。如果我们希望这个函数也包含拆解部分，我们可以使用 `t.Cleanup` 注册一个清理函数：

```go
func TestMySQLIntegration(t *testing.T) {
    // ...
    db := createConnection(t, "tcp(localhost:3306)/db")
    // ...
}

func createConnection(t *testing.T, dsn string) *sql.DB {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        t.FailNow()
    }
    t.Cleanup(
        func() {
            _ = db.Close()
        })
    return db
}
```

在测试结束时，将执行提供给 `t.Cleanup` 的闭包。它使将来的单元测试更容易编写，因为它们不负责关闭 `db` 变量。

请注意，我们可以注册多个清理功能。在那种情况下，这些函数将像我们使用 `defer` 一样执行：后进先出。

关于每个包的安装与卸载，我们必须使用 `TestMain` 函数。`TestMain` 的一个简单实现如下：

```go
func TestMain(m *testing.M) {
    os.Exit(m.Run())
}
```

这个特定的函数接受一个 `*testing.M` 参数，该参数公开一个 `Run` 方法来运行所有测试。因此，我们可以用 setup 和 teardown 函数围绕这个调用：

```go
func TestMain(m *testing.M) {
    setupMySQL()
    code := m.Run()
    teardownMySQL()
    os.Exit(code)
}
```

此代码将在所有测试之前启动 MySQL 一次，然后将其拆除。

使用这些实践来添加安装与卸载功能，我们可以为我们的测试配置一个复杂的环境。

