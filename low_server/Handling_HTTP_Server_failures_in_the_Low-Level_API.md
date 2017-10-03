# 处理服务端错误

在初始化或运行 Akka HTTP 服务器时，很多场景都可能发生错误。默认情况下，Akka 将打印这些错误，有时除简单打印外，还需要其他措施，比如关闭 Actor 系统，或通知外部监控等。

在创建、物质化 HTTP 服务器时可能出现多种错误，错误类型从无法启动服务器，到无法反序列化 `HttpRequest`，具体例子包括（从最外层，到最内层）：

* 无法 `bind` 到特定 address/port
* 无法接收新的 `IncommingConnection`，例如，由于操作系统耗尽文件描述符或内存
* 无法处理请求，例如，接收到错误构造的 `HttpRequest`

本节介绍如何处理上述错误，以及其发生的场景。

## `bind` 错误

第一种错误类型是，服务器无法绑定到指定端口，例如当该端口已被占用，或无该端口的权限（比如只有 root 用户可用）。该场景下 *binding future* 将立刻失败，可以通过监听该 `Future` 的完成事件来做出反应：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.Http.ServerBinding
import akka.stream.ActorMaterializer

import scala.concurrent.Future

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
// needed for the future foreach in the end
implicit val executionContext = system.dispatcher

// let's say the OS won't allow us to bind to 80.
val (host, port) = ("localhost", 80)
val serverSource = Http().bind(host, port)

val bindingFuture: Future[ServerBinding] = serverSource
  .to(handleConnections) // Sink[Http.IncomingConnection, _]
  .run()

bindingFuture.failed.foreach { ex =>
  log.error(ex, "Failed to bind to {}:{}!", host, port)
}
```

一旦服务器成功绑定到端口，`Source[IncommingConnection, _]` 就开始运行，并分发新连接。从技术上讲，该 `Source` 也可能发生错误，但只会在极端场景下发生，比如耗尽文件描述符或内存时，此时该 `Source` 无法接收新的连接。在 Akka Streams 中处理错误非常简单，因为错误总是从发生的极端开始，向下流动，一直到最终阶段。

## 连接源错误

下面例子中，我们添加一个自定义的 `GraphStage`（[Custom stream processing](https://doc.akka.io/docs/akka/2.5.6/scala/stream/stream-customize.html)）以便对流失败做出反应。我们将流失败的原因通知 `failureMonitor` Actor，并让 Actor 决定如何处理 -- 可能重启服务器或关闭 Actor 系统，然而我们不再关心。

```scala
import akka.actor.ActorSystem
import akka.actor.ActorRef
import akka.http.scaladsl.Http
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.Flow

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
implicit val executionContext = system.dispatcher

import Http._
val (host, port) = ("localhost", 8080)
val serverSource = Http().bind(host, port)

val failureMonitor: ActorRef = system.actorOf(MyExampleMonitoringActor.props)

val reactToTopLevelFailures = Flow[IncomingConnection]
  .watchTermination()((_, termination) => termination.failed.foreach {
    cause => failureMonitor ! cause
  })

serverSource
  .via(reactToTopLevelFailures)
  .to(handleConnections) // Sink[Http.IncomingConnection, _]
  .run()
```

## 连接错误

第 3 中失败类型是，连接建立后，突然被中断，例如客户端关闭底层的 TCP 连接。可以用与上面类似的方式处理，并将其作用于连接的 `Flow`：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.Flow

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
implicit val executionContext = system.dispatcher

val (host, port) = ("localhost", 8080)
val serverSource = Http().bind(host, port)

val reactToConnectionFailure = Flow[HttpRequest]
  .recover[HttpRequest] {
    case ex =>
      // handle the failure somehow
      throw ex
  }

val httpEcho = Flow[HttpRequest]
  .via(reactToConnectionFailure)
  .map { request =>
    // simple streaming (!) "echo" response:
    HttpResponse(entity = HttpEntity(ContentTypes.`text/plain(UTF-8)`, request.entity.dataBytes))
  }

serverSource
  .runForeach { con =>
    con.handleWith(httpEcho)
  }
```

上述错误或多或少都与基础设施有关，它们会导致绑定或连接失败。大多数场景下，无需关注其太多细节，因为 Akka 日志总会打印它们，对此类错误，该默认行为是合理的。

若要了解如何处理真实路由层的异常，参考 [Exception Handling](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/exception-handling.html)，该章介绍了如何处理路由异常，并将其转换为 `HttpResponse` 中的错误码和错误提示信息。
