# 客户端如何处理流式 HTTP 实体

## 消费 HTTP 响应实体（客户端）

消费响应实体是最常见的用户场景，可以通过运行 `HttpEntity` 的 `dataBytes` 方法实现。

Akka HTTP 鼓励使用各种流式技术，以充分利用底层的基础设施，例如通过将流入数据分成帧，然后按行解析，最后将 flow 连接到另外一个目标 `Sink`（文件或其他 Akka Streams 连接器）。

```scala
import java.io.File

import akka.actor.ActorSystem
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.{ FileIO, Framing }
import akka.util.ByteString

implicit val system = ActorSystem()
implicit val dispatcher = system.dispatcher
implicit val materializer = ActorMaterializer()

val response: HttpResponse = ???

response.entity.dataBytes
  .via(Framing.delimiter(ByteString("\n"), maximumFrameLength = 256))
  .map(transformEachLine)
  .runWith(FileIO.toPath(new File("/tmp/example.out").toPath))

def transformEachLine(line: ByteString): ByteString = ???
```

然而，有时需要以 `Strict` 形式处理整个实体（即实体将完全载入内存），Akka HTTP 提供了特殊的 `toStrict(timeout)` 方法，用于 eagerly 消费实体，并将其载入内存。

```scala
import scala.concurrent.Future
import scala.concurrent.duration._

import akka.actor.ActorSystem
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import akka.util.ByteString

implicit val system = ActorSystem()
implicit val dispatcher = system.dispatcher
implicit val materializer = ActorMaterializer()

case class ExamplePerson(name: String)
def parse(line: ByteString): ExamplePerson = ???

val response: HttpResponse = ???

// toStrict to enforce all data be loaded into memory from the connection
val strictEntity: Future[HttpEntity.Strict] = response.entity.toStrict(3.seconds)

// while API remains the same to consume dataBytes, now they're in memory already:
val transformedData: Future[ExamplePerson] =
  strictEntity flatMap { e =>
    e.dataBytes
      .runFold(ByteString.empty) { case (acc, b) => acc ++ b }
      .map(parse)
  }
```

## 丢弃 HTTP 响应实体（客户端）

有时调用 HTTP 服务，并不关心响应实体的内容（比如只关心状态码），但前面解释过，无论如何，必须消费实体，否则将对底层的 TCP 连接施加背压。

当不关心实体内容时，可用 `discardEntityBytes` 方法丢弃实体，该方法将流入的数据导入 `Sink.ignore`（译者注：也是某种形式的消费）。

如下两段代码是等价的，也可用于服务端，用于处理进入的 HTTP 请求：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.model.HttpMessage.DiscardedEntity
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer

implicit val system = ActorSystem()
implicit val dispatcher = system.dispatcher
implicit val materializer = ActorMaterializer()

val response1: HttpResponse = ??? // obtained from an HTTP call (see examples below)

val discarded: DiscardedEntity = response1.discardEntityBytes()
discarded.future.onComplete { done => println("Entity discarded completely!") }
```

对应功能等价的底层抽象代码：

```scala
val response1: HttpResponse = ??? // obtained from an HTTP call (see examples below)

val discardingComplete: Future[Done] = response1.entity.dataBytes.runWith(Sink.ignore)
discardingComplete.onComplete(done => println("Entity discarded completely!"))
```
