# 流与 HTTP

Akka HTTP 服务器构建在 [Akka Streams](https://doc.akka.io/docs/akka/2.4.19/scala/stream/index.html) 之上，且其实现、各层 API 都充分利用了 Akka Streams。

Akka HTTP 在连接层提供的接口与 [Working with streaming IO](https://doc.akka.io/docs/akka/2.5.6/scala/stream/stream-io.html) 相同：套接字绑定被表示为进入连接的流。应用从该流中获取连接，并为其中每个连接提供一个 `Flow[HttpRequest, HttpResponse, _]`，该 `Flow` 用于将请求转换为响应。

除将套接字表示为 `Source[IncomingConnection]`，将连接表示为目的地为 `Sink[HttpResponse]` 的 `Source[HttpRequest]` 外，流抽象将 HTTP 请求、响应的实体都表示为 `Source[ByteString]`。查看 [HTTP 模型](../common_abstractions/HTTP_Model.md) 一节，了解更多 HTTP 消息建模的内容。
