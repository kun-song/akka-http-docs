# 自动元组提取

[基础](./Basics.md) 和 [组合指令](./Composing_Directives.md) 两节介绍的路由 DSL 内部是以 `tuple` 进行提取的，例如：

```scala
val futureOfInt: Future[Int] = Future.successful(1)
val route =
  path("success") {
    onSuccess(futureOfInt) {  // Directive[Tuple1[Int]]
      i => complete("Future was completed.")
    }
  }
```

上面例子中的 `onSuccess(futureOfInt)` 返回一个 `Directive1[Int] = Directive[Tuple1[Int]]`。

```scala
val futureOfTuple2: Future[Tuple2[Int, Int]] = Future.successful((1, 2))
val route =
  path("success") {
    onSuccess(futureOfTuple2) {  // Directive[Tuple2[Int, Int]]
      (i, j) => complete("Future was completed.")
    }
  }
```

类似，`onSuccess(futureOfTuple2)` 返回一个 `Directive1[Tuple2[Int, Int]] = Directive[Tuple1[Tuple2[Int, Int]]]`，但它将被自动转换为 `Directive[Tuple2[Int, Int]]` 以避免元组嵌套。

```scala
val futureOfUnit: Future[Unit] = Future.successful(())
val route =
  path("success") {
    onSuccess(futureOfUnit) {  // Directive0
      complete("Future was completed.")
    }
  }
```

当返回 `Future[Unit]` 时，有点特殊，因为这时 `onSuccess(futureOfUnit)` 返回的是 `Directive0`。上面的例子中，`onSuccess(futureOfUnit)` 返回一个 `Directive1[Unit] = Directive[Tuple1[Unit]]`，但 DSL 将 `Unit` 转换为 `Tuple0`，并将结果自动转换为 `Directive[Unit] = Directive0`。
