## 2.16 不使用代码检查工具

linter 是分析代码和捕获错误的自动工具。本节的范围不是提供现有 linters 的详尽列表；否则，它将很快被弃用。然而，我们应该理解并记住为什么 linter 对大多数 Go 项目至关重要。

为了理解为什么代码检查工具很重要，让我们举一个具体的例子。在 *意外变量阴影* 中，我们讨论了与变量阴影有关的潜在误差。使用 `vet`，Go 工具集的标准 linter 和阴影，我们可以检测 `shadow` 变量：

```go
package main

import "fmt"

func main() {
        i := 0
        if true {
                i := 1
                fmt.Println(i)
        }
        fmt.Println(i)
}
```

`vet` 包含在Go二进制文件旁边，让我们先安装 `shadow` ，将其与Go `vet` 链接，然后在前面的示例上运行它。

```shell
$ go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow
$ go vet -vettool=$(which shadow)
./main.go:8:3: declaration of "i" shadows declaration at line 6
```

正如我们所注意到的，`vet` 通知我们，变量 `i` 在本例中被阴影。因此，使用适当的代码检查工具可以帮助我们的代码更健壮，并检测潜在的错误。

> **Note** 代码检查工具没有涵盖这本书的所有错误。因此，建议读者继续他的进步;)。

同样，本节的目标不是列出所有可用的代码检查工具。但是，如果您不是linters的常规用户，以下是您可能希望日常使用的列表：

* [vet](https://golang.org/cmd/vet/) 标准Go分析工具
* [kisielk/errcheck](https://github.com/kisielk/errcheck) 错误检查工具
* [fzipp/gocyclo](https://github.com/fzipp/gocyclo) 循环复杂度分析工具
* [goconst](https://github.com/jgautheron/goconst) 重复字符串常量分析工具

除了 linter，我们还应该使用代码格式化程序来修复代码样式：
* [gofmt](https://golang.org/cmd/gofmt/) 标准Go代码格式化工具
* [goimports](https://pkg.go.dev/golang.org/x/tools/cmd/goimports) 标准Go导入格式化工具

同时，我们也应该看看 `golangci-lint`。这是一个代码检查工具工具，在许多有用的代码检查工具和格式化程序上提供了一个外墙。此外，它允许并行运行代码检查工具，以提高分析速度，这非常方便。

代码检查工具和格式化程序是提高我们代码库质量和一致性的有力方法。让我们花点时间了解我们应该使用哪一个，并确保自动执行它们（例如，CI 或 Git 预提交钩子）。