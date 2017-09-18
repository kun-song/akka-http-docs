# 请求层客户端 API

要使用 Akka HTTP 的客户端功能，推荐使用请求层 API。请求层 API 建立在[主机层 API](./Host-Level_Client-Side_API.md) 之上，提供了获取 HTTP 响应的简单、易用的方法。根据个人喜好，可以采用 flow-based 或者 future-based 两种变体。

>{% em color="red" %}注意{% endem %}

>建议先阅读 [请求、响应实体的流式本质](../streaming_nature/introduction.md) 一节，该节讲解了底层的全栈流的概念，这对来自非 **streaming first** HTTP 客户端背景的用户可能有点出乎意料。

>{% em %}注意{% endem %}

>请求层 API 建立在 actor 系统内部共享的连接池之上，使用连接池带来的后果是，长时间执行的请求将阻塞连接，并导致其他连接处于饥饿状态。不要将请求层 API 用于长时间执行的请求，比如长轮询请求，该类请求可使用 [连接层 API](./Connection-Level_Client-Side_API.md) 或者提供专门的连接池。

## Future-based 变体

一般而言，HTTP 客户端的需求都很简单，只需要获取特定请求的 HTTP 响应即可，并不需要大张旗鼓地建立完善的流式设施。

对于该类场景，Akka HTTP 提供了 `Http().singleRequest()` 方法，用于完成 `HttpRequest` 实例到 `Future[HttpResponse]` 实例的转换。内部实现上，根据请求的 URI，将其分派到（被缓存的）主机连接池上。

此时，请求要么包含绝对 URI 路径，要么包含有效的 `Host` 头部，否则将返回错误。

### 例子

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer

import scala.concurrent.Future

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()

val responseFuture: Future[HttpResponse] =
  Http().singleRequest(HttpRequest(uri = "http://akka.io"))
```

### 在 Actors 中使用 Future-based API

在 `Actor` 中使用基于 `Future` 的 API 时，要谨慎处理 `Future` 的完成。例如，不要在 `Future` 的回调函数（如 `map` `onComplete`）中访问 `Actor` 的状态，应该使用 `pipeTo` 模式将结果作为消息发送给 `Actor`：

```scala
import akka.actor.{ Actor, ActorLogging }
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.stream.{ ActorMaterializer, ActorMaterializerSettings }
import akka.util.ByteString

class Myself extends Actor
  with ActorLogging {

  import akka.pattern.pipe
  import context.dispatcher

  final implicit val materializer: ActorMaterializer = ActorMaterializer(ActorMaterializerSettings(context.system))

  val http = Http(context.system)

  override def preStart() = {
    http.singleRequest(HttpRequest(uri = "http://akka.io"))
      .pipeTo(self)
  }

  def receive = {
    case HttpResponse(StatusCodes.OK, headers, entity, _) =>
      entity.dataBytes.runFold(ByteString(""))(_ ++ _).foreach { body =>
        log.info("Got response, body: " + body.utf8String)
      }
    case resp @ HttpResponse(code, _, _, _) =>
      log.info("Request failed, response code: " + code)
      resp.discardEntityBytes()
  }

}
```

>{% em %}注意{% endem %}

> 确保消费响应实体流（类型为 `Source[ByteString, Unit]`），例如将其导入 `Sink`（若不需要实体内容，可使用 `response.discardEntityBytes()`），否则 Akka HTTP 会将此视为背压信号，并停止读取 TCP 连接。

## Flow-based 变体

基于 `Flow` 的请求层 API 使用 `Http().superPool(...)` 方法表示，该方法创建一个 *super connection pool flow*，根据请求的有效 URI 分别将其路由至（缓存的）主机连接池。

`Http().superPool(...)` 方法返回的 `Flow` 与[主机层客户端 API](./Host-Level_Client-Side_API.md)返回的非常类似，因此，[使用连接池](./Host-Level_Client-Side_API.md#使用主机连接池)一节也适用于此。

主机层 API 提供的 *host connection pool client flow* 与请求层 API 提供的 *super-pool flow* 之间的重要区别为：

* 前者的 `Flow` 包含隐式的目标主机，因此请求本身不需要携带绝对 URI 路径或 `Host` 头部，在需要时 *host connection pool* 会自动添加 `Host` 头部。
* 后者中，所有请求必须携带绝对 URI 或有效的 `Host` 头部，否则无法确定请求的来源。
