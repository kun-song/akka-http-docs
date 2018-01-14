# 基础

我们使用 `Directive` 创建 [Route]()，为了理解指令的工作原理，可以将其与“原始的”路由创建方式比较。

因为 `Route` 仅仅是函数 `RequestContext => Future[RouteResult]` 的类型同义词，所以 `Route` 实例可以以任何合法的函数实例形式出现，例如作为函数字面值：

```scala
val route: Route = { ctx => ctx.complete("yeah") }
```

或者更简短一点：

```scala
val route: Route = _.complete("yeah")
```

而使用 `complete` 指令，可以变得更简短：

```scala
val route = complte("yeah")
```

以上三种路由实现方式完全等价，它们在任何场景下都具有相同的行为。

下面是一个更加复杂的例子，展示了一个重要关注点：

```scala
val a: Route = {
  println("MARK")
  ctx => ctx.complete("yeah")
}

val b: Route = { ctx =>
  println("MARK")
  ctx.complete("yeah")
}
```

路由 `a` 和 路由 `b` 的区别在于 `println` 语句的执行时间：

* 路由 `a` 中 `println` 只会在 `a` 创建时执行一次；
* 路由 `b` 中 `println` 在路由创建时不会执行，而是在路由每次运行时都会执行一次；

使用 `complete` 指令重新实现上面的例子：

```scala
val a = {
  println("MARK")
  complete("yeah")
}

val b = complete {
  println("MARK")
  "yeah"
}
```

重新实现后的路由 `a` `b` 与其原始版本行为相同，因为 `complete` 的参数是以 `by-name` 方式计算的，所以每次路由 `run` 的时候，参数都会被 **重新计算**。

看一个更加复杂的例子：

```scala
val route: Route = { ctx =>
  if (ctx.request.method == HttpMethod.GET)
    ctx.complete("Received GET")
  else
    ctx.complete("Received something else")
}
```

使用 `get` 和 `complete` 指令，可以将上述例子重写如下，当然其行为与之前一致：

```scala
val route =
  get {
    complete("Received GET")
  } ~
  complete("Received something else")
```

注意，如果你愿意，也可以混用这两种不同的路由创建风格：

```scala
val route =
  get { ctx =>
    ctx.complete("Received GET")
  } ~
  complete("Received something else")
```

此处，`get` 指令的内部路由使用显式的函数字面值实现。

从上面的例子可以看出，相对手动创建，使用指令创建路由更加准确，可读性、可维护性更好。而且，指令方式的 **可组合** 性更好（下一节将会看到）。所以使用 Akka HTTP 的路由 DSL 时，一般不要使用手动创建 `Route` 函数，进而直接操作 `RequestContext` 的方式。
