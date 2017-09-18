# 客户端 WebSocket 支持

客户端 WebSocket 支持由 `Http().singleWebSocketRequest`、`Http().webSocketClientFlow` 和 `Http().webSocketClientLayer` 方法提供。

一个 WebSocket 包含两个消息流：入消息（`Sink`）和出消息（`Source`），任意端都可以被发送消息；在连接生命周期中，可以只向一端发送消息。因此 WebSocket 连接可以建模为 `Flow[Message, Message, Mat]` 的连接目标，或者一个需要添加 `Source[Message, Mat]` 和 `Sink[Message, Mat]` 的 `Flow[Message, Message, Mat]`。

WebSocket 请求起始于一个包含 `Upgrade` 头部的 HTTP 请求（当然也可以包含其他 HTTP 请求属性），因此除消息流以外，还有一个服务端发送的初始响应，即 `WebSocketUpgradeResponse`。

当连接成功时，WebSocket 客户端会把连接升级为 WebSocket 连接，并物质化 WebSocket 流。若链接失败，比如收到 `404 NotFound` 错误，则该响应可以在 `WebSocketUpgradeResponse.response` 中找到。

>注意

>确保阅读并理解 [Half-Closed WebSockets](https://doc.akka.io/docs/akka-http/current/scala/http/client-side/websocket-support.html#half-closed-client-websockets)，因为 WebSocket 用于单路通信时的行为，可能出乎你的意料。

## 消息

通过 WebSocket 收发的消息可以是 `TextMessage` 或者 `BinaryMessage`，二者都有两个子类：`Strict`（所有数据在一个块）或 `Streamed`。典型应用中，消息一般是 `Strict`，因为 WebSocket 一般使用短消息（small messages）通信，而非流式数据，但协议本身是允许流式的（[RFC 6455 section 5.2](https://tools.ietf.org/html/rfc6455#section-5.2)）。

*strict text* 可以从 `TextMessage.Strict` 获取，而 *strict binary data* 可以从 `BinaryMessage.Strict` 获取。

流式消息使用 `BinaryMessage.Streamed` 和 `TextMessage.Streamed`，此时，文本、二进制的数据分为作为 `Source[ByteString, NotUsed]` 和 `Source[String, NotUsed]`。

## `singleWebSocketRequest`

`singleWebSocketRequest` 接受 `WebSocketRequest` 和一个 `Flow`（用于连接 WebSocket 连接的 `Source` 和 `Sink`）作为参数。该方法将立即触发请求，并返回一个 *tuple*，其中包含 `Future[WebSocketUpgradeResponse]` 和参数 `Flow` 中的物质化之后的值。

返回的 `Future` 在连接成功建立、服务端正常返回 HTTP 响应时，成功；当连接因异常而失败时，该 `Future` 失败。

下面是发送消息，并打印接受到的消息的简单例子：

```scala
import akka.actor.ActorSystem
import akka.{ Done, NotUsed }
import akka.http.scaladsl.Http
import akka.stream.ActorMaterializer
import akka.stream.scaladsl._
import akka.http.scaladsl.model._
import akka.http.scaladsl.model.ws._

import scala.concurrent.Future

object SingleWebSocketRequest {
  def main(args: Array[String]) = {
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    import system.dispatcher

    // print each incoming strict text message
    val printSink: Sink[Message, Future[Done]] =
      Sink.foreach {
        case message: TextMessage.Strict =>
          println(message.text)
      }

    val helloSource: Source[Message, NotUsed] =
      Source.single(TextMessage("hello world!"))

    // the Future[Done] is the materialized value of Sink.foreach
    // and it is completed when the stream completes
    val flow: Flow[Message, Message, Future[Done]] =
      Flow.fromSinkAndSourceMat(printSink, helloSource)(Keep.left)

    // upgradeResponse is a Future[WebSocketUpgradeResponse] that
    // completes or fails when the connection succeeds or fails
    // and closed is a Future[Done] representing the stream completion from above
    val (upgradeResponse, closed) =
      Http().singleWebSocketRequest(WebSocketRequest("ws://echo.websocket.org"), flow)

    val connected = upgradeResponse.map { upgrade =>
      // just like a regular http request we can access response status which is available via upgrade.response.status
      // status code 101 (Switching Protocols) indicates that server support WebSockets
      if (upgrade.response.status == StatusCodes.SwitchingProtocols) {
        Done
      } else {
        throw new RuntimeException(s"Connection failed: ${upgrade.response.status}")
      }
    }

    // in a real application you would not side effect here
    // and handle errors more carefully
    connected.onComplete(println)
    closed.foreach(_ => println("closed"))
  }
}
```

WebSocket 请求可以包含额外的头部，比如 HTTP Basic Auth：

```scala
val (upgradeResponse, _) =
  Http().singleWebSocketRequest(
    WebSocketRequest(
      "ws://example.com:8080/some/path",
      extraHeaders = Seq(Authorization(
        BasicHttpCredentials("johan", "correcthorsebatterystaple")))),
    flow)
```

## `webSocketClientFlow`

`webSocketClientFlow` 方法接受请求作为参数，并返回 `Flow[Message, Message, Future[WebSocketUpgradeResponse]]`。

该 `Future` 在连接成功建立、服务端正常返回 HTTP 响应时，成功；当连接因异常而失败时，该 `Future` 失败。

>注意

>该方法返回的 `Flow` 只能物质化一次，对于每次请求，必须重新调用该方法以获取新的 `Flow`。

下面是发送消息，并打印接受到的消息的简单例子：

```scala
import akka.actor.ActorSystem
import akka.Done
import akka.http.scaladsl.Http
import akka.stream.ActorMaterializer
import akka.stream.scaladsl._
import akka.http.scaladsl.model._
import akka.http.scaladsl.model.ws._

import scala.concurrent.Future

object WebSocketClientFlow {
  def main(args: Array[String]) = {
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    import system.dispatcher

    // Future[Done] is the materialized value of Sink.foreach,
    // emitted when the stream completes
    val incoming: Sink[Message, Future[Done]] =
      Sink.foreach[Message] {
        case message: TextMessage.Strict =>
          println(message.text)
      }

    // send this as a message over the WebSocket
    val outgoing = Source.single(TextMessage("hello world!"))

    // flow to use (note: not re-usable!)
    val webSocketFlow = Http().webSocketClientFlow(WebSocketRequest("ws://echo.websocket.org"))

    // the materialized value is a tuple with
    // upgradeResponse is a Future[WebSocketUpgradeResponse] that
    // completes or fails when the connection succeeds or fails
    // and closed is a Future[Done] with the stream completion from the incoming sink
    val (upgradeResponse, closed) =
      outgoing
        .viaMat(webSocketFlow)(Keep.right) // keep the materialized Future[WebSocketUpgradeResponse]
        .toMat(incoming)(Keep.both) // also keep the Future[Done]
        .run()

    // just like a regular http request we can access response status which is available via upgrade.response.status
    // status code 101 (Switching Protocols) indicates that server support WebSockets
    val connected = upgradeResponse.flatMap { upgrade =>
      if (upgrade.response.status == StatusCodes.SwitchingProtocols) {
        Future.successful(Done)
      } else {
        throw new RuntimeException(s"Connection failed: ${upgrade.response.status}")
      }
    }

    // in a real application you would not side effect here
    connected.onComplete(println)
    closed.foreach(_ => println("closed"))
  }
}
```

## `webSocketClientLayer`

与普通 HTTP 请求类似（参考 [Stand-Alone HTTP Layer Usage](https://doc.akka.io/docs/akka-http/current/scala/http/client-side/connection-level.html#http-client-layer)），WebSocket 也与底层 TCP 接口完全解耦，且其应用场景与普通 HTTP 请求类似。

返回的 HTTP 层组成一个 `BidiFlow[Message, SslTlsOutbound, SslTlsInbound, Message, Future[WebSocketUpgradeResponse]]`。

## 半关闭 WebSocket

Akka HTTP 的 WebSocket API 不支持半关闭连接，即任意一端流完成后，整改连接即关闭（”关闭握手”之后或 3s 超时已过）。这可能导致意料之外的行为，比如若仅消费服务端发送的消息，如下：

```scala
// we may expect to be able to to just tail
// the server websocket output like this
val flow: Flow[Message, Message, NotUsed] =
  Flow.fromSinkAndSource(
    Sink.foreach(println),
    Source.empty)

Http().singleWebSocketRequest(
  WebSocketRequest("ws://example.com:8080/some/path"),
  flow)
```

由于流物质化后，`Source.empty` 会立刻完成，从而立即关闭连接。为了避免该场景，可以使用 `Source.maybe`：

```scala
// using Source.maybe materializes into a promise
// which will allow us to complete the source later
val flow: Flow[Message, Message, Promise[Option[Message]]] =
  Flow.fromSinkAndSourceMat(
    Sink.foreach[Message](println),
    Source.maybe[Message])(Keep.right)

val (upgradeResponse, promise) =
  Http().singleWebSocketRequest(
    WebSocketRequest("ws://example.com:8080/some/path"),
    flow)

// at some later time we want to disconnect
promise.success(None)
```

这会阻止 `Source` 完成，但不会发送任何元素，直到手动关闭 `Promise`，从而完成 `Source` 并关闭连接。

当发送有限个数的元素时，也会发生同样的问题，因为最后一个元素到达之后，`Source` 会立刻关闭，并导致连接连接。为避免该问题，同样可以使用 `Source.maybe`：

```scala
// using emit "one" and "two" and then keep the connection open
val flow: Flow[Message, Message, Promise[Option[Message]]] =
  Flow.fromSinkAndSourceMat(
    Sink.foreach[Message](println),
    Source(List(TextMessage("one"), TextMessage("two")))
      .concatMat(Source.maybe[Message])(Keep.right))(Keep.right)

val (upgradeResponse, promise) =
  Http().singleWebSocketRequest(
    WebSocketRequest("ws://example.com:8080/some/path"),
    flow)

// at some later time we want to disconnect
promise.success(None)
```

WebSocket 中常见场景及解决方案：

| 场景 | 可用方案     |
| Two-way communication | `Flow.fromSinkAndSource`, or `Flow.map` for a request-response protocol       |
| Infinite incoming stream, no outgoing | `Flow.fromSinkAndSource(someSink, Source.maybe)` |
| Infinite outgoing stream, no incoming | `Flow.fromSinkAndSource(Sink.ignore, yourSource)` |
