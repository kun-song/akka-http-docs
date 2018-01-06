# RequestContext

`RequestContext` 包裹 `HttpRequest` 实例，并添加路由所需的额外信息，如 `ExecutionContext`，`Materializer`，`LoggingAdapter` 和 `RoutingSettings`。另外还包含 `unmatchedPath`，其值描述了请求 URL 与 [Path Directives](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/directives/path-directives/index.html) 当前匹配多少。

`RequestContext` 是不可变类型，但提供辅助函数，用于方便创建修改的副本。
