# 指令的类型安全性

使用 `|` 和 `&` 操作符组合指令时，路由 DSL 将保证所有 **提取** 行为与预期一致，且编译时将强加逻辑约束。

例如，若一个指令会提取一个值，而另一个执行不提取任何值，则不能使用 `|` 操作符连接两者：

```scala
val route = path("order" / IntNumber) | get // 无法编译
```

并且，提取值的 **数量** 和 **类型** 必须匹配：

```scala
val route = path("order" / IntNumber) | path("order" / DoubleNumber) // 无法编译
val route = path("order" / IntNumber) | parameter('order.as[Int]) // OK
```

使用 `&` 组合有提取行为的指令时，所有的提取值将被妥善处理：

```scala
val order = path("order" / IntNumber) & parameter('oem, 'expired ?)
val route =
  order { (id, oem, expired) => ...}
```

借助指令，可以从小的构建块开始，以可插拔、有趣的方式构建 web 服务的逻辑，并且保持 DRY 以及类型安全。若预定义的指令无法满足需求，也可以很容易地的创建自定义指令。
