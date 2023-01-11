## 7.5 重复处理了两次错误

多次处理错误是开发人员经常犯的错误，而不是专门在 Go 中。让我们了解为什么这是一个问题以及如何有效地处理错误。

为了说明这个问题，让我们编写一个 `GetRoute` 函数来获取从一对源到一对目标坐标的路线。假设此函数将调用未导出的 `getRoute` 函数，其中包含计算最佳路线的业务逻辑。在调用 `getRoute` 之前，我们必须使用 `validateCoordinates` 验证源坐标和目标坐标。我们还希望记录可能的错误。这是一个可能的实现：

```go
func GetRoute(srcLat, srcLng, dstLat, dstLng float32) (Route, error) {
    err := validateCoordinates(srcLat, srcLng)
    if err != nil {
        log.Println("failed to validate source coordinates")
        return Route{}, err
    }

    err = validateCoordinates(dstLat, dstLng)
    if err != nil {
        log.Println("failed to validate target coordinates")
        return Route{}, err
    }

    return getRoute(srcLat, srcLng, dstLat, dstLng)
}

func validateCoordinates(lat, lng float32) error {
    if lat > 90.0 || lat < -90.0 {
        log.Printf("invalid latitude: %f", lat)
        return fmt.Errorf("invalid latitude: %f", lat)
    }
    if lng > 180.0 || lng < -180.0 {
        log.Printf("invalid longitude: %f", lng)
        return fmt.Errorf("invalid longitude: %f", lng)
    }
    return nil
}
```

该代码有什么问题？首先，我们可以注意到在 `validateCoordinates` 中重复 `invalid latitude` 或 `invalid longitude` 错误消息在日志记录和返回的错误中是多么麻烦。此外，如果我们以无效纬度运行它，例如，它将记录以下行：

```shell
2021/06/01 20:35:12 invalid latitude: 200.000000
2021/06/01 20:35:12 failed to validate source coordinates
```

因此，一个错误有两个日志行，这是一个问题。为什么？因为它使调试变得更加困难。例如，如果该函数被并发调用多次，那么这两条消息在日志中可能不会一个接一个，从而使调试过程更加复杂。

根据经验，错误应该只处理一次。记录错误就是处理错误，返回错误也是如此。因此，我们应该记录或返回错误，而不是两者兼而有之。

让我们重写我们的实现以只处理一次错误：

```go
func GetRoute(srcLat, srcLng, dstLat, dstLng float32) (Route, error) {
    err := validateCoordinates(srcLat, srcLng)
    if err != nil {
        return Route{}, err
    }

    err = validateCoordinates(dstLat, dstLng)
    if err != nil {
        return Route{}, err
    }

    return getRoute(srcLat, srcLng, dstLat, dstLng)
}

func validateCoordinates(lat, lng float32) error {
    if lat > 90.0 || lat < -90.0 {
        return fmt.Errorf("invalid latitude: %f", lat)
    }
    if lng > 180.0 || lng < -180.0 {
        return fmt.Errorf("invalid longitude: %f", lng)
    }
    return nil
}
```

在这个版本中，每个错误只处理一次，直接返回。然后，假设 `GetRoute` 的调用者正在处理可能的日志错误，如果纬度无效，它将输出以下消息：

```shell
2021/06/01 20:35:12 invalid latitude: 200.000000
```

这个新的 Go 版本的代码完美吗？还不是最完美。例如，第一个实现在无效纬度的情况下返回两个日志。不过，我们知道哪个对 `validateCoordinates` 的调用失败了：源坐标或目标坐标。在这里，我们丢失了这些信息。因此，我们需要为错误添加额外的上下文。

让我们使用 Go 1.13 错误包装重写最新版本的代码（我们将省略 `validateCoordinates`，因为它将保持不变）：

```go
func GetRoute(srcLat, srcLng, dstLat, dstLng float32) (Route, error) {
    err := validateCoordinates(srcLat, srcLng)
    if err != nil {
        return Route{},
                fmt.Errorf("failed to validate source coordinates: %w", err)
    }

    err = validateCoordinates(dstLat, dstLng)
    if err != nil {
        return Route{},
                fmt.Errorf("failed to validate target coordinates: %w", err)
    }

    return getRoute(srcLat, srcLng, dstLat, dstLng)
}
```

`validateCoordinates` 返回的每个错误现在都被包装为错误提供额外的上下文，无论它与源坐标还是目标坐标相关。因此，如果我们运行这个新版本，以下是调用者在源纬度无效的情况下将记录的内容：

```shell
2021/06/01 20:35:12 failed to validate source coordinates:
invalid latitude: 200.000000
```

处理错误应该只进行一次。正如我们所见，记录错误就是处理错误。因此，我们应该记录或返回错误。通过这样做，我们可以简化代码并更好地了解错误情况。使用错误包装是最方便的方法，因为它允许我们传播源错误并将上下文添加到错误中。

在下一节中，我们将看到在 Go 中忽略错误的适当方法。