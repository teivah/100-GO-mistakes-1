## 4.3 在 `range` 循环中使用指针元素被忽略的影响

本节将深入探讨在使用带有指针元素的 `range` 循环时的一个特定错误。如果我们不够谨慎，可能会导致我们引用错误元素的问题。让我们了解这个问题以及如何解决它。

在开始之前，让我们澄清一下使用指针元素的切片或映射的基本原理？主要有以下三种场景：
* 在语义方面，使用指针语义存储数据意味着共享元素。例如，使用以下方法保存在缓存中插入元素的逻辑：

```go
type Store struct {
    m map[string]*Foo
 }
 
 func (s Store) Put(id string, foo *Foo) {
    s.m[id] = foo
    // ...
 }
```

在这里，使用指针语义意味着 `Foo` 元素由 `Put` 的调用者和 `Store` 结构共享。

* 有时，我们已经在操作指针。因此，直接存储在我们的集合指针中而不是值中会很方便。
* 另一种情况，如果我们存储大型结构并且这些结构经常发生突变，为了避免每个突变的复制和插入，我们可以使用指针来代替：

```go
func updateMapValue(mapValue map[string]LargeStruct, id string) {
            value := mapValue[id] 
            value.foo = "bar"
            mapValue[id] = value 
    }
    
    func updateMapPointer(mapPointer map[string]*LargeStruct, id string) {
            mapPointer[id].foo = "bar" 
    }
```

由于 `updateMapPointer` 接受指针映射，因此 `foo` 字段的突变可以在一个步骤中完成。

现在，是时候讨论 `range` 循环中指针元素的常见错误了。我们将考虑以下两个结构：
* 代表消费者的 `Customer` 结构。
* 一个包含 `Customer` 指针映射的 `Store` 结构体。

```go
type Customer struct {
        ID      string
        Balance float64
}

type Store struct {
        m map[string]*Customer
}
```

同时，我们将考虑以下方法，该方法遍历一部分 `Customer` 元素并将它们存储到 `m` 映射中：
```go
func (s *Store) storeCustomers(customers []Customer) {
        for _, customer := range customers {
                s.m[customer.ID] = &customer
        }
}
```

在此示例中，我们使用 `range` 运算符迭代输入切片并将 `Customer` 指针存储到映射中。但是，这种方法能达到我们的预期吗？

让我们通过使用三个不同 `Customer` 结构的切片调用它来尝试一下：

```go
s.storeCustomers([]Customer{
        {ID: "1", Balance: 10},
        {ID: "2", Balance: -10},
        {ID: "3", Balance: 0},
})
```

如果我们打印map，这是这段代码的结果：

```go
key=1, value=&main.Customer{ID:"3", Balance:0}
key=2, value=&main.Customer{ID:"3", Balance:0}
key=3, value=&main.Customer{ID:"3", Balance:0}
```

我们可以注意到，不是存储三个不同的 `Customer` ，而是存储在 map 中的所有元素都引用相同的 `Customer` 结构：3 。 那么我们做错了什么？

当我们使用 `range` 循环遍历客户切片时，无论元素数量如何，它都会创建一个具有固定地址的客户变量。我们可以通过在每次迭代期间打印指针地址来验证这一点：

```go
func (s *Store) storeCustomers(customers []Customer) {
        for _, customer := range customers {
                fmt.Printf("%p\n", &customer)
                        s.m[customer.ID] = &customer
        }
}
```

```shell
0xc000096020
0xc000096020
0xc000096020
```

它为什么如此重要？让我们检查在每次迭代期间如何：
* 在第一次迭代期间，`custormer` 引用第一个元素：`Custormer 1`。我们存储了一个 `custormer` 的指针。
* 在第二次迭代中，`custormer` 现在引用另一个元素：`Customer 2`。我们也存储了一个 `custormer` 的指针。
* 最后，在最后一次迭代中，`custormer` 引用了最后一个元素：`Customer 3`。同样，相同的指针存储在map中。

![](https://img.exciting.net.cn/33.png)

在迭代结束时，我们在映射中存储了三次相同的指针。这个指针的最后一个赋值是对切片最后一个元素 `Custormer 3` 的引用：这就是为什么映射的所有元素都引用相同的 `Custormer`。

那么我们如何解决这个问题呢？有两个主要的解决方案。

第一个类似于我们在 *Unintended variable shadowing* 中看到的；它需要创建一个局部变量：

```go
func (s *Store) storeCustomers(customers []Customer) {
        for _, customer := range customers {
                current := customer
                s.m[current.ID] = &current
        }
}
```

在此示例中，我们不存储引用的 `customer` 指针，但 `current.current` 是在每次迭代期间引用唯一 `Customer` 的变量。因此，在循环之后，我们在映射中存储了引用不同 `Customer` 结构的不同指针。

另一种解决方案是使用切片索引存储引用每个元素的指针：

```go
func (s *Store) storeCustomers(customers []Customer) {
        for i := range customers {
                customer := &customers[i]
                        s.m[customer.ID] = customer
        }
}
```

在这个解决方案中，`customer` 现在是一个指针。由于它在每次迭代期间都被初始化，因此它具有唯一的地址。因此，我们将在映射中存储不同的指针。

当使用 `range` 循环遍历数据结构时，我们必须记住，所有值都分配给具有单个唯一地址的唯一变量。因此，如果我们在每次迭代期间存储一个引用该变量的指针，我们最终会出现存储引用相同元素的相同指针的情况：最新的一个。我们可以通过强制在循环范围内创 建局部变量或创建通过索引引用切片元素的指针来克服这个问题。两种解决方案都很好。另请注意，我们将切片数据结构作为输入，但问题与map类似。

在下一节中，我们将看到与地图迭代相关的常见错误。