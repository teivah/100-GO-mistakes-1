## 11.4 表驱动测试，简化测试代码的神器

表驱动测试是一种简化编写测试代码的有效技术，它减少了模式化的代码，帮助我们更加专注于重要的事情：测试逻辑。本节将通过一个具体示例来展示为什么在使用 Go 时表驱动测试值得了解。

让我们思考以下从字符串中删除所有新行后缀（`\n` 或 `\r\n`）的函数：

```go
func removeNewLineSuffixes(s string) string {
    if s == "" {
        return s
    }
    if strings.HasSuffix(s, "\r\n") {
        return removeNewLineSuffixes(s[:len(s)-2])
    }
    if strings.HasSuffix(s, "\n") {
        return removeNewLineSuffixes(s[:len(s)-1])
    }
    return s
}
```

此函数以递归方式删除所有 `\r\n` 或 `\n` 的后缀。现在，假设我们要广泛地测试这个函数。我们至少应该涵盖以下几种情况：

* 输入为空
* 输入以 `\n` 结尾
* 输入以 `\r\n` 结尾
* 输入以多个 `\n` 结尾
* 输入结束没有换行符

一种可能的方法是为每个案例创建一个单元测试：

```go
func TestRemoveNewLineSuffix_Empty(t *testing.T) {
    got := removeNewLineSuffixes("")
    expected := ""
    if got != expected {
        t.Errorf("got: %s", got)
    }
}

func TestRemoveNewLineSuffix_EndingWithCarriageReturnNewLine(t *testing.T) {
    got := removeNewLineSuffixes("a\r\n")
    expected := "a"
    if got != expected {
        t.Errorf("got: %s", got)
    }
}

func TestRemoveNewLineSuffix_EndingWithNewLine(t *testing.T) {
    got := removeNewLineSuffixes("a\n")
    expected := "a"
    if got != expected {
        t.Errorf("got: %s", got)
    }
}

func TestRemoveNewLineSuffix_EndingWithMultipleNewLines(t *testing.T) {
    got := removeNewLineSuffixes("a\n\n\n")
    expected := "a"
    if got != expected {
        t.Errorf("got: %s", got)
    }
}

func TestRemoveNewLineSuffix_EndingWithoutNewLine(t *testing.T) {
    got := removeNewLineSuffixes("a\n")
    expected := "a"
    if got != expected {
        t.Errorf("got: %s", got)
    }
}
```

每个函数都代表我们想要涵盖的特定案例。但是，我们可以注意到两个主要缺点。首先，函数名称变得更加复杂（`TestRemoveNewLineSuffix_EndingWithCarriageReturnNewLine` 的长度为 55 个字符），因此它会迅速影响函数应该测试的内容的清晰度。第二个缺点是这些函数之间的重复量，因为结构总是相同的：

* 调用 `removeNewLineSuffixes`
* 定义期望值
* 比较值
* 记录错误消息

如果我们想更改这些步骤之一，例如，将预期值作为错误消息的一部分，我们将不得不在所有测试中重复它。而且我们编写的测试越多，维护就越困难。

由于这些原因，我们可以使用表驱动测试，这样我们只编写一次逻辑。表驱动测试依赖于子测试，这意味着单个测试函数可以包含多个子测试。例如，以下测试包含两个子测试：

```go
func TestFoo(t *testing.T) {
    t.Run("subtest 1", func(t *testing.T) {
        if false {
            t.Error()
        }
    })
    t.Run("subtest 2", func(t *testing.T) {
        if 2 != 2 {
            t.Error()
        }
    })
}
```

`TestFoo` 函数包括两个子测试。如果我们运行这个测试，它会显示`subtest 1` 和 `subtest 2` 的结果：

```shell
--- PASS: TestFoo (0.00s)
--- PASS: TestFoo/subtest_1 (0.00s)
--- PASS: TestFoo/subtest_2 (0.00s)
PASS
```

我们还可以使用 `‑run` 标志运行单个测试，并将父测试名称与子测试连接起来。例如，仅运行 `subtest 1`：

```shell
$ go test -run=TestFoo/subtest_1 -v
=== RUN   TestFoo
=== RUN   TestFoo/subtest_1
--- PASS: TestFoo (0.00s)
--- PASS: TestFoo/subtest_1 (0.00s)
```

现在让我们回到我们的示例，看看如何利用子测试来防止重复测试逻辑。主要思想是为每个案例创建一个子测试。存在不同的变体，但我们将讨论一个映射数据结构，其中键代表测试名称，值代表测试数据（预期的输入）。

使用包含测试数据的数据结构并利用子测试来避免样板代码是表驱动测试的概念。这是使用映射的实现：

```go
func TestRemoveNewLineSuffix(t *testing.T) {
    tests := map[string]struct {
        input    string
        expected string
    }{
        `empty`: {
            input:    "",
            expected: "",
        },
        `ending with \r\n`: {
            input:    "a\r\n",
            expected: "a",
        },
        `ending with \n`: {
            input:    "a\n",
            expected: "a",
        },
        `ending with multiple \n`: {
            input:    "a\n\n\n",
            expected: "a",
        },
        `ending without newline`: {
            input:    "a",
            expected: "a",
        },
    }
    for name, tt := range tests {
        t.Run(name, func(t *testing.T) {
            got := removeNewLineSuffixes(tt.input)
            if got != tt.expected {
                t.Errorf("got: %s, expected: %s", got, tt.expected)
            }
        })
    }
}
```

`tests` 是一个映射类型的变量。key 是测试名称，值代表测试数据；在我们的例子中：输入和预期的字符串。每个映射条目都是我们想要涵盖的新测试用例。然后，我们为每个映射条目运行一个新的子测试。

这个测试解决了我们讨论的两个缺点：

* 每个测试名称现在都是一个字符串，而不是 PascalCase 函数名称，使其更易于阅读。
* 逻辑只写一次，并为所有不同的情况共享。修改测试结构或添加新测试只需要很少的修改工作。

关于表驱动程序测试的最后一件事也可能是错误的来源：正如我们之前所讨论的，我们可以通过调用 `t.Parallel` 来标记并行运行的测试。我们也可以在提供给 `t.Run` 的闭包内的子测试中执行此操作：

```go
for name, tt := range tests {
    t.Run(name, func(t *testing.T) {
        t.Parallel()
        // Use tt
    })
}
```

然而，我们应该注意到这个闭包使用了一个循环变量。因此，为了防止在不小心使用 goroutines 和循环变量中讨论类似的错误，导致闭包可能使用错误的 `tt` 变量值，我们应该创建另一个变量或隐藏它：

```go
for name, tt := range tests {
    tt := tt
    t.Run(name, func(t *testing.T) {
        t.Parallel()
        // Use tt
    })
}
```

这样，每个闭包都将访问自己的 `tt` 变量。

总之，我们应该记住，如果多个单元测试具有相似的结构，我们可以使用表驱动测试将它们简化。由于它可以防止重复，因此可以轻松更改测试逻辑并更容易添加新用例。

让我们开始讨论如何防止 Go 中的易碎测试。