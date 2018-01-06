# 路由

路由是 Akka HTTP 路由 DSL 的核心概念。使用路由 DSL 构建的结构，无论只有一行，还是多达几百行，都是下面类型的实例：

```scala
type Route = RequestContext => Future[RouteResult]
```

`Route` 为函数，将 `RequestContext` 转化为 `Future[RouteResult]`。

通常，当路由接收到请求（即 `RequestContext`）时，会做：

1. 完成请求，通过返回 `requestContext.complete(...)` 的值；
2. 拒绝请求，通过返回 `requestContext.reject(...)` 的值；
3. 失败请求，通过返回 `requestContext.fail(...)` 的值，或直接抛出异常（参考 [Exception Handling](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/exception-handling.html)）；
4. 执行异步处理，立即返回 `Future[RouteResult]`；

第一种场景很清晰，调用 `complete`，发送请求对应的响应到客户端；第二种场景中，“拒绝”意味着路由不想处理该请求。

可用 `Route.seal` *seals* `Route`，但需要 `RejectionHandler` 和 `ExceptionHandler` 将拒绝、异常转换为合适的 HTTP 响应。查看 [Sealing a Route](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/routes.html#sealing-a-route) 获取更多细节。

使用 `Route.handlerFlow` 或 `Route.asyncHandler`，可将 `Route` 提升为 *handler flow* 或 *async handler function*，进一步用于 `bindAndHandleXXX` 函数。

注意：`RouteResult` 伴生对象中定义了从 `Route` 到 `Flow[HttpRequest, HttpResponse, Unit]` 的隐式转换，该转换依赖 `Route.handlerFlow`。
