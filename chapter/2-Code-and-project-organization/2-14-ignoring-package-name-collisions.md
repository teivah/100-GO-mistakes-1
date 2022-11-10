## 2.14 忽略软件包名称冲突

当变量名与现有软件包名称碰撞阻止软件包被重用时，就会发生软件包冲突。让我们看看一个库暴露 Redis 客户端的具体例子：

```go
package redis

type Client struct { ... }

func NewClient() *Client { ... }

func (c *Client) Get(key string) (string, error) { ... }
```

现在，让我们跳到 client。尽管软件包名称称为 `redis`，但它在 Go 中完全有效，也可以创建一个名为 `redis` 的变量：

```go
redis := redis.NewClient()
v, err := redis.Get("foo")
```

在这里，`redis` 变量名与 `redis` 包冲突。即使这是允许的，也应该避免。事实上，在整个 `redis` 变量范围内， `redis` 软件包不再可访问。此外，假设限定符在整个函数中引用变量和包名。在这种情况下，代码阅读器知道限定符指的是什么可能是模棱两可的。有哪些选择来避免这种碰撞？

第一个选项是找到一个不同的变量名。例如：

```go
redisClient := redis.NewClient()
v, err := redisClient.Get("foo")
```

这可能是最直截了当的方法。但是，如果出于某种原因，我们更喜欢继续命名我们的变量 redis，我们包的导入做文章。

使用软件包导入，我们可以使用别名更改限定符以引用 `redis` 软件包。例如：

```go
import redisapi "mylib/redis"

// ...

redis := redisapi.NewClient()
v, err := redis.Get("foo")
```

在这里，我们使用 `redisapi` 导入别名来引用 `redis` 包，以便我们可以继续命名我们的变量 `redis` 。

> **Note** 一种选择也可以是使用点导入来访问软件包的所有公共元素，而无需软件包限定符。然而，这种方法往往会增加混乱，在大多数情况下，应该避免。

我们还请注意，我们还应该避免命名变量和内置函数之间的冲突：

`copy := copyFile(src, dst)`

在这种情况下，只要 `copy` 变量存在，`copy` 内置函数就无法访问。

总之，我们应该防止变量名称冲突，以避免歧义。如果我们面临冲突，我们应该要么找到另一个有意义的名称，要么使用导入别名。

在下一节中，我们将看到一个与代码文档相关的常见错误。