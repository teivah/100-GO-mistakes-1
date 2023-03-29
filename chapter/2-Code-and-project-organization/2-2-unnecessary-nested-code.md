## 2.2 不必要的嵌套代码

应用于软件的核心模型是系统行为的内部表示。在编程时，我们必须不断维护核心模型，例如关于整体代码交互和功能实现的核心模型。基于命名、一致性、格式等多个标准，代码是否具有可读性。可读代码将需要较少的认知努力来维护核心模型；因此它会更容易阅读和维护。

可读性的一个关键方面是嵌套级别的数量。让我们做一个练习。假设您在一个新项目上工作，并且必须了解以下函数 `join` 函数的作用：

```go
func join(s1, s2 string, max int) (string, error) {
    if s1 == "" {
        return "", errors.New("s1 is empty")
    } else {
        if s2 == "" {
            return "", errors.New("s2 is empty")
        } else {
            concat, err := concatenate(s1, s2)
            if err != nil {
                return "", err
            } else {
                if len(concat) > max {
                    return concat[:max], nil
                } else {
                    return concat, nil
                }
            }
        }
    }
}
```

此函数连接两个字符串，如果长度大于 `max` 则返回一个子字符串。同时，它处理对 `s1`、`s2` 的检查以及对 `concatenate` 的调用是否返回错误。从实现的角度来看，这个函数是正确的。然而，建立一个涵盖所有不同案例的核心模型，可能不是一项简单的任务。为什么？由于嵌套级别的数量。

现在，让我们再次尝试使用相同的功能进行此练习，但实现方式不同：

```go
func join(s1, s2 string, max int) (string, error) {
    if s1 == "" {
        return "", errors.New("s1 is empty")
    }
    if s2 == "" {
        return "", errors.New("s2 is empty")
    }
    concat, err := concatenate(s1, s2)
    if err != nil {
        return "", err
    }
    if len(concat) > max {
        return concat[:max], nil
    }
    return concat, nil
}

func concatenate(s1 string, s2 string) (string, error) {
    // ...
}
```

您可能应该已经注意到，构建这个新版本的心智模型需要更少的认知负荷，尽管做的工作和以前一样。然而，这里我们只维护两个嵌套级别。正如 Go Time 播客的小组成员 Mat Ryer 所说：

> 将快乐的路径向左对齐；您应该能够快速向下扫描一列以查看预期的执行流程。

由于嵌套的 `if/else` 语句，很难区分第一个版本中的预期执行流程。相反，第二个版本需要向下扫描一列以查看预期的执行流程，并向下扫描第二列以查看边缘情况的处理方式，如下图所示：

![](https://img.exciting.net.cn/24.png)

一般来说，一个函数需要的嵌套层数越多，阅读和理解就越复杂。让我们看看这条规则的不同应用来优化我们的代码的可读性：

当 `if` 块在所有情况下都返回时，我们应该省略 `else` 块。我们不应该这样写：

```go
if foo() {
    // ...
    return true
} else {
    // ...
}
```

但是，请省略 `else` 块，如下所示：

```go
if foo() {
    return true
}
// ...
```

在这个新版本中，以前存在于 `else` 中的代码块被移到顶层，使其更易于阅读。

* 我们也可以按照这个逻辑走一条不愉快的路：

```go
if s != "" {
    // ...
} else {
    return errors.New("empty string")
}
```

这里，`s` 为空表示非快乐路径。因此，我们应该像这样翻转条件：

```go
if s == "" {
    return errors.New("empty string")
}
// ...
```

这个新版本更容易阅读，因为它保留了左边缘的快乐路径并减少了块的数量。

编写可读的代码对每个开发人员来说都是一项重要的挑战。努力减少嵌套块的数量，对齐左边的快乐路径，尽早返回是提高代码可读性的具体手段。

在下一节中，我们将讨论 Go 项目中的一个常见误用：init 函数。
