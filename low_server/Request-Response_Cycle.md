# 请求、响应循环

新请求被接收后，会被发布为一个 `Http.IncomingConnection`，其中包含远端地址，以及获取 `Flow[HttpRequest, HttpResponse, _]`（用于处理该连接进入的请求）的方法。

通过 `handleWithXXX` 方法处理请求，该方法接受一个 handler 作为参数：

* `Flow[HttpRequest, HttpResponse, _]` 用于 `handleWith`
* 函数 `HttpRequest => HttpResponse` 用于 `handleWithSyncHandler`
* 函数 `HttpRequest => HttpResponse` 用于 `handleWithAsyncHandler`

下面是一个完整的例子：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model.HttpMethods._
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.Sink

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
implicit val executionContext = system.dispatcher

val serverSource = Http().bind(interface = "localhost", port = 8080)

val requestHandler: HttpRequest => HttpResponse = {
  case HttpRequest(GET, Uri.Path("/"), _, _, _) =>
    HttpResponse(entity = HttpEntity(
      ContentTypes.`text/html(UTF-8)`,
      "<html><body>Hello world!</body></html>"))

  case HttpRequest(GET, Uri.Path("/ping"), _, _, _) =>
    HttpResponse(entity = "PONG!")

  case HttpRequest(GET, Uri.Path("/crash"), _, _, _) =>
    sys.error("BOOM!")

  case r: HttpRequest =>
    r.discardEntityBytes() // important to drain incoming HTTP Entity stream
    HttpResponse(404, entity = "Unknown resource!")
}

val bindingFuture: Future[Http.ServerBinding] =
  serverSource.to(Sink.foreach { connection =>
    println("Accepted new connection from " + connection.remoteAddress)

    connection handleWithSyncHandler requestHandler
    // this is equivalent to
    // connection handleWith { Flow[HttpRequest] map requestHandler }
  }).run()
```

该例中，通过 `HttpRequest => HttpResponse` 函数与 `handleWithSyncHandler` 将请求转换为响应。根据场景不同，可以选用不同的 handler。若应用使用 `Flow` 处理请求，则应用对于每个请求，必须恰好产生一个响应，而且必须保证响应顺序与请求顺序匹配（HTTP pipeling），若使用 `handleWithSyncHandler` 或 `handleWithAsyncHandler` 或 `map` `mapAsync` 操作子，则自动满足这些要求。

使用 [Routing DSL] 可以更容易创建请求 handler。

### 流式请求/响应实体

通过 `HttpEntity` 子类支持流式的 HTTP 消息实体。当接收请求或构造响应时，应用需要能够处理流式实体。查阅 [HttpEntity](../common_abstractions/HTTP_Model.md) 获取更多信息。

若使用 Akka HTTP 提供的序列化、反序列化设施，则自定义类型与流式实体之间的转换非常方便。

### 关闭连接

当 `Flow` 取消订阅，或者连接一方关闭连接时，HTTP 连接都将被关闭。在 `HttpResponse` 中显式添加 `Connection: close` 头部可以很方便关闭连接，这将是该连接上最后一个响应，该响应发出后，服务器将主动关闭连接。

当请求实体被取消（`Sink.cancelled()`），或者被部分消费（使用 `take` 组合子）时，连接也会关闭。为避免该场景，需要将实体连到 `Sink.ignore()` 上，以便显式耗尽实体。
