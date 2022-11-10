## 2.15 缺少代码文档

文档是编码的一个重要方面。它简化了客户端如何使用 API，但也可以帮助维护项目。在 Go 中，我们应该遵循一些规则，使我们的代码模式化。让我们深入研究它们。

首先，必须描述每个对外公开的元素。无论是结构、接口、函数等。如果已对外公开，则必须记录在案。惯例是从导出元素的名称开始发表评论。例如：

```go
// Customer is a customer representation.
type Customer struct{}

// ID returns the customer identifier.
func (c Customer) ID() string { return "" }
```

作为一个惯例，每个注释都应该是一个完整的句子，以标点符号结尾。我们还请记住，当我们注释一个函数（或方法）时，我们应该强调函数打算做什么，而不是它是如何做到的；这属于函数和注释的核心，而不是文档。此外，理想情况下，文档应该提供足够的信息，让 client 不深入研究我们的代码，以了解如何使用导出的元素。

> **Note** 然后，如果开发人员使用 `ComputePath` 函数，他应该有一个警告（大多数 IDE 处理不建议使用的注释）。
>
> 可以使用 `// Deprecated：`以这种方式注释来弃用导出的元素：

```go
// ComputePath returns the fastest path between two points.
// Deprecated: This function uses a deprecated way to compute
// the fastest path. Use ComputeFastestPath instead.
func ComputePath() {}
```

当谈到记录变量或常量时，我们可能有兴趣传达两个方面：其目的和内容。前者应该作为代码文档，对外部客户端有用。然而，后者不一定是公开的。例如：

```go
// DefaultPermission is the default permission used by the store engine.
const DefaultPermission = 0o644 // Need read and write accesses.
```

此常量表示默认权限。代码文档传达了其目的，而常量旁边的注释描述了其实际内容（读写访问）。

为了帮助客户和维护者了解软件包的范围，我们还应该记录每个软件包。惯例是以 `// Package` 开头注释，然后以这种方式添加软件包名称：

```go
// Package math provides basic constants and mathematical functions.
//
// This package does not guarantee bit-identical results
// across architectures.
package math
```

软件包注释的第一行应简明扼要，因为它将显示在软件包列表中：

![](https://img.exciting.net.cn/6.png)

然后，我们可以在以下行中提供我们需要的所有信息。

注释软件包可以在该软件包的任何 Go 文件中完成；没有规则。一般来说，我们应该尝试将其放入与软件包相同名称的相关文件中或特定文件（如 `doc.go` ）。

关于一揽子文件，最后要提到的一件事是，与声明不相邻的注释被省略。例如，以下版权注释将不会出现在生成的文档中：

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package math provides basic constants and mathematical functions.
//
// This package does not guarantee bit-identical results
// across architectures.
package math
```

总之，我们应该记住，每个导出的元素都必须记录在案。记录我们的代码不应该成为限制。我们应该借此机会确保它能帮助客户和维护者了解我们代码的目的。

在本章的最后一节中，我们将看到一个关于工具的常见错误：不使用代码检查工具。