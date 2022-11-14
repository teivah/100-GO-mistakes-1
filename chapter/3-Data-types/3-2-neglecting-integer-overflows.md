## 3.2 忽略整数溢出

不了解在 Go 中如何处理整数溢出可能会导致关键错误。本节将深入探讨这个问题。但首先，让我们提醒我们一些与整数有关的概念。

### 3.2.1 概念

Go 总共提供了十种整数类型。有四种有符号整数类型：

* `int8`: 8 bits
* `int16`: 16 bits
* `int32`: 32 bits
* `int64`: 64 bits

和四种无符号整数类型：

* `uint8`: 8 bits
* `uint16`: 16 bits
* `uint32`: 32 bits
* `uint64`: 64 bits

此外，两种整数类型是最常用的：

* `int`
* `uint`

这两种类型的大小取决于系统：32位系统为32位，64位系统为64位。

现在，我们来讨论一下溢出。我们将使用 `int32`，我们将初始化到其最大值并增加它。此代码的行为应该是什么？

```go
var counter int32 = math.MaxInt32
counter++
fmt.Printf("counter=%d\n", counter)
```

此代码在运行时编译且不会崩溃。但是，`counter++` 语句将生成一个整数溢出：

```go
counter=-2147483648
```

一个当算术运算术运算创建值时，就会发生整数溢出在可以表示为给定数量的范围内之外字节。

一个 `int32` 类型的变量使用32位表示。以下是最大 `int32` 值（`math.MaxInt32`）的二进制表示形式：

```go
01111111111111111111111111111111
 |------31 bits set to 1-------|
```

`int32` 是一个有符号整数，左边的位代表整数的符号：正数为0，负数为1。如果我们加一这个整数，有没有空位来表示新值。因此，它将导致一个整数溢出。二进制，以下是新值：

```go
10000000000000000000000000000000
 |------31 bits set to 0-------|
```

我们可以注意到，位符号现在等于1，意思是负数。这值是表示的有符号32位整数的最小值。

> **Note** 最小的负值不是111111111111111111111111111111111。 事实上，大多数系统依靠两者的互补操作来表示二进制数字（反转每个位并加1）。主要目标：此操作是使x-x等于零，无论x。

在 Go 中，可以在编译时检测到的整数溢出将生成编译错误。例如：

```go
var counter int32 = math.MaxInt32 + 1
```

```go
constant 2147483648 overflows int32
```

然而，在运行时，整数溢出和下流是静默的（没有提示的）；它不会导致应用程序崩溃。必须在编码时就意识到，因为它可能会导致偷偷摸摸的错误——例如，整数自增或添加导致负结果的正整数。

在深入研究如何使用常见操作检测整数溢出之前，让我们考虑一下何时需要担心它。在大多数情况下，比如处理请求的计数器或基本的加法/乘法，如果我们使用正确的整数类型，我们不应该太担心。然而，在某些特定情况下，例如使用较小整数类型的内存受限项目、处理大数字或进行转换时，我们可能需要检查可能的溢出。

> **Note** Ariane 5 火箭发射失败是由于将 64 位浮点数转换为 16 位有符号整数导致的溢出。

### 3.2.2 在自增期间检测整数溢出

如果我们想在基于定义大小（int8、int16、int32、int64、uint8、uint16、uint32 或 uint64）的类型的递增操作期间检测整数溢出，可以通过根据 `math` 常量值来实现 常数。 例如，使用 int32：

```go
func Inc32(counter int32) int32 {
        if counter == math.MaxInt32 {
                panic("int32 overflow")
        }
        return counter + 1
}
```

此函数检查输入是否已经等于 `math.MaxInt32`。我们知道，如果是这样的话，增量会导致溢出。

`Int` 和 `uint` 类型呢？在Go 1.17之前，我们必须手动构建这些常量。现在，它们是 `math` 包的一部分：`math.MaxInt`、`math.MinInt` 和 `math.MaxUint`。

如果我们必须在 `int` 类型上测试溢出，我们可以使用 `math.MaxInt`：

```go
func IncInt(counter int) int {
        if counter == math.MaxInt {
                panic("int overflow")
        }
        return counter + 1
}
```

`Uint` 的逻辑是一样的；我们可以使用 `math.MaxUint`：

```go
func IncUint(counter uint) uint {
        if counter == math.MaxUint {
                panic("uint overflow")
        }
        return counter + 1
}
```

我们已经看到了如何在增量操作后检查整数溢出。现在，添加呢？

### 3.2.3 在加法期间检测整数溢出

我们如何检测加法期间的整数溢出？通过重用 `math.MaxInt`：

```go
func AddInt(a, b int) int {
        if a > math.MaxInt-b {
                panic("int overflow")
        }

        return a + b
}
```

`a` 和 `b` 是两个操作数；如果 `a` 大于 `math.MaxInt - b`，这意味着操作将导致整数溢出。

现在，让我们看看最新的操作：乘法。

### 3.2.4 在乘法期间检测整数溢出

乘法处理起来有点复杂。我们必须对最小整数进行检查：`math.MinInt`：

实施情况如下：

```go
func MultiplyInt(a, b int) int {
        if a == 0 || b == 0 {
                return 0
        }

        result := a * b
        if a == 1 || b == 1 {
                return result
        }
        if a == math.MinInt || b == math.MinInt {
                panic("integer overflow")
        }
        if result/b != a {
                panic("integer overflow")
        }
        return result
}
```

用乘法检查整数溢出需要多个步骤。 首先，测试其中一个操作数是否等于 0、1 或 `math.MinInt`。 然后我们将乘法结果除以b。 如果结果不等于原始因子 (a)，则表示发生了整数溢出。

总的来说，整数溢出（和下溢）是 Go 中的静默操作。 如果我们想检查溢出是否导致溢出以避免偷偷摸摸的错误，我们可以使用本节中描述的实用程序函数。 我们还要提醒一下，Go 提供了一个处理大数的包：`math/big`。 如果 `int` 不够用，它也可能是一个选项。

让我们在下一节中继续讨论基本的 Go 类型和浮点。

