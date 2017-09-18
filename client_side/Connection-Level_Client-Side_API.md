# 连接层客户端 API

连接层 API 是 Akka HTTP 中最底层的客户端 API，可以完全控制连接的打开、关闭时机，请求到连接的分发逻辑。最强的灵活性，代价是最差的易用性。

>{% em color="red" %}注意{% endem %}

>建议先阅读 [请求、响应实体的流式本质](../streaming_nature/introduction.md) 一节，该节讲解了底层的全栈流的概念，这对来自非 **streaming first** HTTP 客户端背景的用户可能有点出乎意料。

## 打开 HTTP 连接

借助连接层 API，通过物质化 `Http().outgoingConnection(...)` 返回的 `Flow`，可以打开针对特定端点的连接，示例如下：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import akka.stream.scaladsl._

import scala.concurrent.Future
import scala.util.{ Failure, Success }

object WebClient {
  def main(args: Array[String]): Unit = {
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    implicit val executionContext = system.dispatcher

    val connectionFlow: Flow[HttpRequest, HttpResponse, Future[Http.OutgoingConnection]] =
      Http().outgoingConnection("akka.io")

    def dispatchRequest(request: HttpRequest): Future[HttpResponse] =
      // This is actually a bad idea in general. Even if the `connectionFlow` was instantiated only once above,
      // a new connection is opened every single time, `runWith` is called. Materialization (the `runWith` call)
      // and opening up a new connection is slow.
      //
      // The `outgoingConnection` API is very low-level. Use it only if you already have a `Source[HttpRequest]`
      // (other than Source.single) available that you want to use to run requests on a single persistent HTTP
      // connection.
      //
      // Unfortunately, this case is so uncommon, that we couldn't come up with a good example.
      //
      // In almost all cases it is better to use the `Http().singleRequest()` API instead.
      Source.single(request)
        .via(connectionFlow)
        .runWith(Sink.head)

    val responseFuture: Future[HttpResponse] = dispatchRequest(HttpRequest(uri = "/"))

    responseFuture.andThen {
      case Success(_) => println("request succeeded")
      case Failure(_) => println("request failed")
    }.andThen {
      case _ => system.terminate()
    }
  }
}
```

除主机名字、端口以外，`Http().outgoingConnection(...)` 方法还可以设置连接的套接字选项，以及一些其他配置。

注意只有 `Flow` 物质化之后，才会尝试建立连接！若 `Flow` 物质化多次，则每次物质化都会建立一条 **独立连接**，若连接建立失败，则 **立即停止** 已物质化的 `Flow`，并给出对应的异常。

## 请求-响应循环

`Flow` 一旦物质化，就做好了消费 `HttpRequest` 的准备。请求不断被发送到连接上，而响应则不断从连接进入，并分发到下游流水线。当然，背压将作用于连接的各个部分，即当下游流水线消费响应变慢时，请求源将放缓发送请求的速度。

底层连接上发生的任何错误，都被视为异常，并终止响应流（且取消请求源）。

注意，若源在接收到前面请求的响应之前，继续发送请求，则这些请求将依次发送到连接上（查看 [HTTP_pipelining](https://en.wikipedia.org/wiki/HTTP_pipelining)），并非所有服务器都支持该特性。若在接收到全部请求的响应之前，服务端关闭连接，则会以截断错误关闭响应流。

## 关闭连接

当接收到的响应包含 `Connection: close` 头部时，Akka HTTP 将主动关闭连接；服务端也可以关闭连接。

通过完成请求流，应用可以主动触发连接关闭，该场景下，当接受完最后一个挂起的响应后，底层 TCP 连接将被关闭。

响应实体被取消（将其导入 `Sink.cancelled()`）或部分消费（例如使用 `take` 组合子）时，也会关闭连接。为 {% em %}避免{% endem %} 该场景，应该显式将实体导入 `Sink.ignore()`。

## 超时

目前 Akka HTTP 没有实现客户端的请求超时检查，因为该功能可以视为流基础设施的通用特性。

Akka Stream 提供了多种超时功能，比如 `idleTimeout`、`backpressureTimeout`、`completionTimeout`、`initialTimeout` 和 `throttle`，任何使用 Akka Stream 的 API 都可以使用它们。参考 Akka Stream 文档获取更多细节。

更多 Akka HTTP 的超时支持，请参考 [Akka HTTP 超时](../common_abstractions/Akka_HTTP_Timeout.md)。

## 单独的 HTTP 层用法

因为 Akka HTTP 基于 *Reactive-Streams*，所以其与底层 TCP 是完全解耦的。在特定场景中，能够将 HTTP 层用于 **非网络** 数据来源是很有用的，比如测试、调试或底层事件源（e.g by replaying network traffic）。

在客户端侧，单独的 HTTP 层组成 `BidiStage`，它可以将编码的、原始的底层连接升级到 HTTP 层，其定义如下：

```scala
/**
 * The type of the client-side HTTP layer as a stand-alone BidiFlow
 * that can be put atop the TCP layer to form an HTTP client.
 *
 * {{{
 *                +------+
 * HttpRequest  ~>|      |~> SslTlsOutbound
 *                | bidi |
 * HttpResponse <~|      |<~ SslTlsInbound
 *                +------+
 * }}}
 */
type ClientLayer = BidiFlow[HttpRequest, SslTlsOutbound, SslTlsInbound, HttpResponse, NotUsed]
```

通过调用 `Http().clientLayer` 的两个重载方法可以获取 `Http.ClientLayer` 实例，该实例也允许配置。
