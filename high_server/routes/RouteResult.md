# RouteResult

`RouteResult` 是简单的代数数据类型（ADT），`RouteResult` 对 `Route` 的所有未出错结果进行了建模，其定义如下：

```scala
sealed trait RouteResult

object RouteResult {
  final case class Complete(response: HttpResponse) extends RouteResult
  final case class Rejected(rejections: immutable.Seq[Rejection]) extends RouteResult
}
```

一般不需要手动创建 `RouteResult` 实例，而是通过预定义的 `RouteDirectives`（例如 `complete`, `reject` 和 `redirect`）或者 `RequestContext` 定义的相关方法创建。
