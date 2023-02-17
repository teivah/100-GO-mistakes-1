## 10.4 SQL 类库的常见错误

`database/sql` 提供了一个围绕 SQL（或类似 SQL）数据库的通用接口。在使用这个包时发现常见的模式或错误也很常见。让我们深入研究五个常见错误。

### 10.4.1 `sql.Open` 不一定会建立数据库的连接

使用 `sql.Open` 时，一个常见的误解是期望此函数实际上建立与数据库的连接：

```go
db, err := sql.Open("mysql", dsn)
if err != nil {
	return err
}
```

然而，情况不一定如此。根据文档：

> Open 可能只是验证其参数而不创建与数据库的连接。
> 
> -- sql.Open 文档

实际上，行为取决于所使用的 SQL 驱动程序。对于某些驱动程序，`sql.Open` 不建立任何连接；相反，它只是为以后使用做准备（例如，使用 `db.Query`）。因此，可能会延迟建立与 DB 的第一个连接。

为什么了解这种行为很重要？例如，在某些情况下，我们希望仅在知道所有依赖项都已正确设置且可访问后才准备好服务。如果我们不这样做，尽管配置错误，该服务仍可能产生一些流量。

因此，如果我们要保证使用 `sql.Open` 的函数也保证底层 DB 是可达的，我们应该使用 `Ping` 方法：

```go
db, err := sql.Open("mysql", dsn)
if err != nil {
    return err
}
if err := db.Ping(); err != nil {
    return err
}
```

`Ping` 强制建立连接，确保数据源名称有效且数据库可访问。请注意，ping 的替代方法是 `PingContext`，它要求提供额外的上下文来传达 ping 何时应该被取消或超时。
尽管可能违反直觉，但让我们记住 `sql.Open` 不一定建立任何连接，并且可以延迟打开第一个连接。如果我们想要测试我们的配置并且数据库是可访问的，我们应该在 `sql.Open` 之后调用 `Ping` 或 `PingContext` 方法。

### 10.4.2 忘记配置连接池参数

与默认 HTTP 客户端和服务器不提供在生产中可能无效的默认行为相同，了解 Go 中如何处理 DB 连接至关重要。

`sql.Open` 返回一个 `*sql.DB` 结构。该结构不代表单个数据库连接；相反，它代表一个连接池。值得注意的是，我们不想手动实现它。池中的连接可以有两种状态：

* 已经使用（例如，被另一个触发查询的 goroutine 使用）
* 空闲（已创建但暂时未使用）

此外，重要的是要记住创建池也意味着四个可用的配置参数。这些参数中的每一个都是 `*sql.DB` 的导出方法：

* `SetMaxOpenConns`:与数据库的最大打开连接数（默认值：无限制）
* `SetMaxIdleConns`:最大空闲连接数（默认值：2）
* `SetConnMaxIdleTime`:连接关闭前可以空闲的最长时间（默认值：无限制）
* `SetConnMaxLifetime`:连接在关闭之前可以保持打开状态的最长时间（默认值：无限制）

这是一个最大连接数为五个的示例：

![](https://img.exciting.net.cn/70.png)

此示例有四个正在进行的连接：三个空闲和一个已在使用中。因此，剩余一个可用插槽可用于额外连接。如果有新查询进来，它将选择一个空闲连接（如果仍然可用）。如果没有更多空闲连接，如果有额外的可用插槽，池将创建一个新连接；否则，它将等待连接可用。

那么，我们为什么要调整这些配置参数呢？

* 设置 `SetMaxOpenConns` 对于生产环境的应用程序很重要。事实上，由于默认值是无限的，我们应该设置它以确保它适合底层数据库可用的。
* 如果我们的应用程序生成大量并发请求，则应该适当增加 `SetMaxIdleConns` 的值（默认值为 2）。否则，可能会导致频繁重连。 
* 如果我们的应用程序可能面临大量请求，则设置 `SetConnMaxIdleTime` 很重要。事实上，当应用程序恢复到更平和的状态时，我们希望确保创建的连接最终被释放。 
* 最后但同样重要的是，如果我们连接到负载平衡的数据库服务器，设置 `SetConnMaxLifetime` 也很有用。在这种情况下，我们要确保我们的应用程序永远不会使用连接太久。

对于生产环境的应用，请记住考虑这四个参数。另外，请注意，如果一个应用程序面临不同的用例，我们可以使用多个连接池。

### 10.4.3 推荐使用 prepared 语句

prepared 语句是许多 SQL 数据库实现的功能，用于执行重复的 SQL 语句。在内部，SQL 语句是预先编译的，并与提供的数据分开。有两个主要好处：

* 效率：语句不用重新编译（编译就是解析+优化+翻译）
* 安全性：降低 SQL 注入攻击的风险

因此，如果一个语句被重复，我们应该使用 prepared 语句。同时，我们还应该在不受信任的上下文中使用 prepared 语句（例如，在 Internet 上公开请求参数映射到 SQL 语句的参数）。

要使用 prepared 语句，我们必须调用 `Prepare`方法:`*sql.DB`，而不是调用 `Query` 方法：

```go
stmt, err := db.Prepare("SELECT * FROM ORDER WHERE ID = ?")
if err != nil {
    return err
}
rows, err := stmt.Query(id)
// ...
```

首先，我们使用 prepared 语句，然后在提供参数的同时执行它。`Prepare` 方法的第一个输出是 `*sql.Stmt`，可以重复使用和并发运行。我们还要注意，当不再需要该语句时，必须使用 `Close()` 方法将其关闭。

> **Note** `Prepare` 和 `Query` 方法还可以提供额外的上下文：`PrepareContext` 和 `QueryContext`。

为了效率和安全性，让我们记住在有意义的时候使用 prepared 语句。

### 10.4.4 错误的处理空值

下一个错误是使用查询错误处理空值。

让我们写一个例子，我们将检索员工的部门和年龄：

```go
rows, err := db.Query("SELECT DEP, AGE FROM EMP WHERE ID = ?", id)
if err != nil {
    return err
}
// Defer closing rows

var (
    department string
    age int
)
for rows.Next() {
    err := rows.Scan(&department, &age)
    if err != nil {
        return err
    }
    // ...
}
```

我们使用 `Query` 来执行查询。然后，我们遍历行并使用 `Scan` 将列复制到 `department` 和 `age` 指针所指向的值中。如果我们运行这个例子，我们可能会在调用 `Scan` 时得到以下错误：

```shell
2021/10/29 17:58:05 sql: Scan error on column index 0, name
"DEPARTMENT": converting NULL to string is unsupported
```

此处，SQL 驱动程序引发错误，因为部门值等于 `NULL`。如果列可以为空，则有两个选项可以防止 `Scan` 返回错误。

第一种方法是将 `department` 声明为字符串指针：

```go
var (
    department *string
    age        int
)
for rows.Next() {
    err := rows.Scan(&department, &age)
    // ...
}
```

我们提供要 `scan` 的是指针的地址，而不是直接提供字符串类型的地址。通过这样做，如果值为 NULL，则 `department` 将为 nil。

另一种方法是使用其中一种 `sql.NullXXX` 类型。例如 `sql.NullString`：

```go
var (
    department sql.NullString
    age        int
)
for rows.Next() {
    err := rows.Scan(&department, &age)
    // ...
}
```

`sql.NullString` 是字符串之上的包装器。它包含两个导出字段：包含字符串值的 `String` 和表示字符串是否不为 NULL 的 `Valid`。可以访问以下包装器：

* `sql.NullString`
* `sql.NullBool`
* `sql.NullInt32`
* `sql.NullInt64`
* `sql.NullFloat64`
* `sql.NullTime`

这两种方法都有效，也许 `sql.NullXXX` 更清楚地表达了意图，正如核心 Go 维护者 Russ Cox 所提到的：

> 没有有效的区别。我们认为人们可能想要使用 NullString，因为它很常见，并且可能比 *string 更清楚地表达了意图。但任何一个都行。
> 
> -- Russ Cox

因此，让我们记住，可空列的最佳实践是将其作为指针处理或使用 `sql.NullXXX` 类型。

### 10.4.5 不处理行迭代错误

另一个常见的错误是在遍历行时遗漏了可能的错误。让我们看看这个错误处理被误用的函数：

```go
func get(ctx context.Context, db *sql.DB, id string) (string, int, error) {
    rows, err := db.QueryContext(ctx,
            "SELECT DEP, AGE FROM EMP WHERE ID = ?", id)
    if err != nil {
        return "", 0, err
    }
    defer func() {
        err := rows.Close()
        if err != nil {
            log.Printf("failed to close rows: %v\n", err)
        }
    }()

    var (
        department string
        age        int
    )
    for rows.Next() {
        err := rows.Scan(&department, &age)
        if err != nil {
            return "", 0, err
        }
    }
    return department, age, nil
}
```

在这个函数中，我们处理了三个错误：执行查询时、关闭行和扫描一行。然而，这还不够。我们必须知道 `for rows.Next() {}` 循环可能会在没有更多行时或在准备下一行时发生错误时中断。在行迭代之后，我们应该调用 `rows.Err` 来区分这两种情况：

```go
func get(ctx context.Context, db *sql.DB, id string) (string, int, error) {
    // ...
    for rows.Next() {
        // ...
    }

    if err := rows.Err(); err != nil {
        return "", 0, err
    }

    return department, age, nil
}
```

这是要牢记的最佳实践：因为 `rows.Next` 可以在我们遍历所有行时或在准备下一行时发生错误时停止，所以我们应该在迭代后检查 
`rows.Err`。

现在让我们讨论一个常见的错误：忘记关闭瞬态资源。