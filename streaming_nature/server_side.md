# 服务端如何处理流式 HTTP 实体

与客户端类似，HTTP 实体直接与流关联，而流由底层 TCP 连接提供。因此，如 **请求实体** 不被消费，则服务端将背压连接，由客户端决定对该连接数据的处理。

注意，部分指令强制隐式 `toStrict` 操作，比如 `entity(as[String])` 以及其他类似指令。

## 消费 HTTP 请求实体（服务端）

消费请求实体最简单的方式为，将其转换为实际的领域对象，比如，可以使用 [`entity`](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/marshalling-directives/entity.html) 指令：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport._
import akka.stream.ActorMaterializer
import spray.json.DefaultJsonProtocol._

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
// needed for the future flatMap/onComplete in the end
implicit val executionContext = system.dispatcher

final case class Bid(userId: String, bid: Int)

// these are from spray-json
implicit val bidFormat = jsonFormat2(Bid)

val route =
  path("bid") {
    put {
      entity(as[Bid]) { bid =>
        // incoming entity is fully consumed and converted into a Bid
        complete("The bid was: " + bid)
      }
    }
  }
```

当然，也可以获取其原始 dataBytes，并运行底层的流，比如将流导入 FileIO Sink，当全部数据写入文件后，通过 `Future[IoResult]` 通知结束：

```scala
import akka.actor.ActorSystem
import akka.stream.scaladsl.FileIO
import akka.http.scaladsl.server.Directives._
import akka.stream.ActorMaterializer
import java.io.File

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
// needed for the future flatMap/onComplete in the end
implicit val executionContext = system.dispatcher

val route =
  (put & path("lines")) {
    withoutSizeLimit {
      extractDataBytes { bytes =>
        val finishedWriting = bytes.runWith(FileIO.toPath(new File("/tmp/example.out").toPath))

        // we only want to respond once the incoming data has been handled:
        onComplete(finishedWriting) { ioResult =>
          complete("Finished writing data: " + ioResult)
        }
      }
    }
  }
```

## 丢弃 HTTP 请求实体（服务端）

有时基于某种验证（比如检查指定用户是否允许上传文件），你可能决定丢弃上传的实体。

注意 *丢弃* 意味着虽然对上传的数据不感兴趣，但上传操作将继续 -- 当仅对指定实体不感兴趣，但不能终止 **整个连接**（后面会讲）时可能有用，因为可能有很多其他请求在该连接上阻塞了。

为了显式丢弃 dataBytes，可以执行 `HttpRequest` 的 `discardEntityBytes` 方法：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.server.Directives._
import akka.stream.ActorMaterializer
import akka.http.scaladsl.model.HttpRequest

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
// needed for the future flatMap/onComplete in the end
implicit val executionContext = system.dispatcher

val route =
  (put & path("lines")) {
    withoutSizeLimit {
      extractRequest { r: HttpRequest =>
        val finishedWriting = r.discardEntityBytes().future

        // we only want to respond once the incoming data has been handled:
        onComplete(finishedWriting) { done =>
          complete("Drained all data from connection... (" + done + ")")
        }
      }
    }
  }
```

相关的概念是 **取消** 进入的 `entity.dataBytes` 流，这会导致 Akka HTTP 服务端突然关闭连接。这在检测到当前用户根本不允许上传，需要关闭连接（而非读取并忽略数据）时有用。将 `entity.dataBytes` 连接到 `Sink.cancelled()` 将取消实体流，且服务端将关闭底层连接，不再接受任何请求：

```scala
import akka.actor.ActorSystem
import akka.stream.scaladsl.Sink
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model.headers.Connection
import akka.stream.ActorMaterializer

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
// needed for the future flatMap/onComplete in the end
implicit val executionContext = system.dispatcher

val route =
  (put & path("lines")) {
    withoutSizeLimit {
      extractDataBytes { data =>
        // Closing connections, method 1 (eager):
        // we deem this request as illegal, and close the connection right away:
        data.runWith(Sink.cancelled) // "brutally" closes the connection

        // Closing connections, method 2 (graceful):
        // consider draining connection and replying with `Connection: Close` header
        // if you want the client to close after this request/reply cycle instead:
        respondWithHeader(Connection("close"))
        complete(StatusCodes.Forbidden -> "Not allowed!")
      }
    }
  }
```

[关闭连接](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/server-side/low-level-api.html#http-closing-connection-low-level) 章节有更详细的解释。

## 待处理：自动丢弃未使用实体

某些情况下，针对指定请求，可以检测到几乎不会被用户使用的实体，此时可以发出告警，或者自动丢弃实体。该高级特性还没有实现。

>注意

>社区建议 Akka HTTP 添加 `auto draining` 特性，我们希望实现、或者帮助社区实现该特性。

>查看 [issue #183](https://github.com/akka/akka-http/issues/183)、[issue #117](https://github.com/akka/akka-http/issues/117) 了解更多，欢迎大家贡献代码！
