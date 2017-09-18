# 主机层客户端 API

与[连接层客户端 API](../client_side/Connection-Level_Client-Side_API.md)相比，主机层 API 免去手动管理每个 HTTP 连接的繁琐，它为每个目标端点（host/port 结合）自动管理一个可配置的连接池。

>{% em color="red" %}注意{% endem %}

>建议先阅读 [请求、响应实体的流式本质](../streaming_nature/introduction.md) 一节，该节讲解了底层的全栈流的概念，这对来自非 **streaming first** HTTP 客户端背景的用户可能有点出乎意料。

## 获取主机连接池

获取给定目标端点的连接池最好的方法为 `Http().cachedHostConnectionPool(...)`，该方法返回 `Flow`，可加入应用级别的流设置，该 `Flow` 称之为 *pool client flow*。

支撑 *pool client flow* 的底层连接池是被缓存的，对于每个 `ActorSystem`、目标端点、连接池设置，任何时间最多只有一个连接池存活。

根据配置，HTTP 层可透明处理连接池的空闲关闭、重启，因此在应用生命周期内，*client flow* 实例保持有效，可以任何频繁的将其 *materialized*，且两次物质化之间的时间间隔无任何限制。

使用 `Http().cachedHostConnectionPool(...)` 获取 *pool client flow* 时，Akka HTTP 将立即启动连接池（甚至在第一个 client flow 物质化之前），但在第一个请求到来之前，该连接池不会自动打开对目标端点的连接。

## 配置主机连接池

除连接层配置、*socket* 选项以外，Akka HTTP 允许配置连接池本身的行为，查阅[配置](../configuration/Configuration.md)章节的 `akka.http.host-connection-pool` 部分，获取更多可用配置的信息。

注意，对于同一个目标主机，若使用不同配置获取连接池，则最终得到各自独立的多个连接池，这意味着针对目标端点，应用打开的并发连接总数，可能会超过单个连接池的 `max-connections` 设置！

`max-open-requests` 配置需要着重讲解下。该配置限制了特定连接池在同一时间可以处理的最大请求数量。若应用调用 `Http().cachedHostConnectionPool(...)` 3 次（相同端点、相同配置），则获取到 {% em %}同一连接池{% endem %} 上的 3 个不同 *client flow* 实例，若每个 *client flow* 被（并发地）物质化 4 次，则应用得到 12 个并发运行的 *client flow materialization*，它们共享同一个连接池。

这意味着，若连接池的 `pipelining-limit` 设置为 1（效果为禁止 *pipelining*），则同一时间最多同时打开 12 个请求；若 `pipelining-limit` 设置为 8，则理论上最多同时打开 96 个请求（8 * 12）。

`max-open-requests` 用来强制限制同时打开的请求数量，主要用于防止错误使用连接池带来的危害。例如，若应用对同一连接池物质化了过多的 *client flow*。

## 使用主机连接池

`Http().cachedHostConnectionPool(...)` 返回的 *pool client flow* 类型如下：

```scala
Flow[(HttpRequest, T), (Try[HttpResponse], T), HostConnectionPool]
```

该方法消费 `(HttpRequest, T)`，得到 `(Try[HttpResponse], T)`，似乎有点过于复杂，在请求、响应两端都包含一个自定义类型 `T` 对象的原因是，底层传输常常建立在多个连接上，因此 *pool client flow* 产生响应的顺序，经常与其处理的请求的顺序不同。当然也可以在分发响应之前，按照请求顺序将响应重排序，但此时单个耗时响应将阻塞其他（可能已经处理完成的）响应。

为了避免此类阻塞，*pool client flow* 会立刻分发已准备好的响应，而不关心请求的顺序，如此一来，就需要一种方式将响应与其对应的请求关联起来。Akka HTTP 允许请求携带一个自定义的 *上下文* 对象，该对象将与响应一起传递回应用，上下文对象的类型 `T` 对 Akka HTTP 是完全不透明的，可以自由选取最适合的类型。

>注意

>使用连接池的缺点是，长时间运行的请求（如长轮询）将阻塞连接，可能饿死其他请求，对于此类请求，使用 [连接层 API](./Connection-Level_Client-Side_API.md)。


## 连接分配逻辑

Akka HTTP 为进入的请求分配连接的逻辑如下：

1. 若存在 **空闲连接**，则将请求分配给该连接；
2. 若无空闲连接，但连接池未满，则创建一个新连接；
3. 若连接池已满，则在仅处理 **幂等请求** 的连接中，选择请求数最少的（`pipelining-limit`）；
4. 否则，对请求来源施加 **背压**，停止接收新请求。

更多关于请求、连接调度的细节，参考 Wikipedia 的 [HTTP pipelining](https://en.wikipedia.org/wiki/HTTP_pipelining) 条目。

## 请求重试

若 `max-retries` 配置大于 0，则对于无法获取响应的 {% em %}幂等请求{% endem %}，Akka HTTP 将执行请求重试。幂等请求是指，根据 HTTP 标准，其 HTTP 方法为幂等方法的请求，目前 Akka HTTP 中除了 `POST` `PATH` `CONNECT` 以外，其他方法皆为幂等方法。

若无法获取请求的响应，可能是如下 3 种错误场景：

1. 请求在到达服务端之前丢失
2. 处理请求时，服务端发生错误
3. 响应在服务端发送之后丢失

由于无法确定是具体是什么原因导致响应不可得，而 `PATCH` 和 `POST` 请求可能已经在服务端触发了 **非幂等动作**，所以该类请求 **不能重试**。

对此类场景，以及重试之后依然无法获取响应的场景，连接池将产生一个失败的 `Try`（例如 `scala.util.Failure`）以及用户自定义的请求上下文。

## 连接池关闭

完成 *pool client flow* 会断开其与连接池的关系，连接池本身将继续运行，因为它（当前，或者未来）可能服务多个 *client flow*。只有超过为连接池配置的 `idle-timeout` 之后，Akka HTTP 才会自动停止连接池，并释放其所有资源。

若使用 `Http().cachedHostConnectionPool(...)` 再次获取 *client flow*，或重新物质化已经存在的 *client flow*，则自动、透明地启动对应的连接池。

除了通过 `idle-timeout` 自动关闭连接池外，还可以通过调用 `HostConnectionPool` 实例的 `shutdown()` 方法 **立即关闭** 连接池，该方法将产生一个 `Future[Unit]`，并在连接池完成关闭后进行填充。

通过在 `ActorSystem` 中调用 `Http().shutdownAllConnectionPools()` 方法，可以立即停止 {% em %}全部{% endem %} 连接池，该方法将产生一个 `Future[Unit]`，并在全部连接池完成关闭后进行填充。

>注意

>如果在关闭 `ActorSystem` 时遇到 `akka.stream.AbruptTerminationException` 异常，请确保在关闭整个系统前，先关闭全部活跃连接，这可以借助 `Http().shutdownAllConnectionPools()` 完成，仅当该方法的 `Future` 完成之后，才关闭整个 `ActorSystem`。

## 例子

>注意

>前面有个示例使用了 `Source.single(request).via(pool).runWith(Sink.head)`，实际上，这是一种 **反模式**，效果并不好。推荐使用 {% em %}请求队列{% endem %} 或 {% em %}流式风格{% endem %}。

### 主机层 API + 队列

在很多场景下，仅需要发出请求，并在完成后接收响应，此时使用 [请求层 API](./Request-Level_Client-Side_API.md) 即可。下面介绍如何结合使用基于 `Future` 的 API 与主机层 API。

Akka HTTP 禁止创建无界缓存、无限连接，这主要通过以下两点实现：

1. 对连接池中的所有请求施加背压
2. 当内部缓存溢出、存在太多 materialization，或者请求太多时，给请求返回 `BufferOverflowException` 异常

为模仿请求层 API，可以在连接池外添加一个显式队列，自定义队列溢出时的处理逻辑，如下例所示（感谢 [kazuhiro’s blog](http://kazuhiro.github.io/scala/akka/akka-http/akka-streams/2016/01/31/connection-pooling-with-akka-http-and-source-queue.html) 提供的最初思路）。

可根据实际内存限制修改 `QueueSize` 值，任意场景下，都应该考虑当队列溢出导致请求失败时，应该采取的处理策略（例如重试，或直接失败）。

```scala
import scala.util.{ Failure, Success }
import scala.concurrent.{ Future, Promise }

import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import akka.stream.scaladsl._

import akka.stream.{ OverflowStrategy, QueueOfferResult }

implicit val system = ActorSystem()
import system.dispatcher // to get an implicit ExecutionContext into scope
implicit val materializer = ActorMaterializer()

val QueueSize = 10

// This idea came initially from this blog post:
// http://kazuhiro.github.io/scala/akka/akka-http/akka-streams/2016/01/31/connection-pooling-with-akka-http-and-source-queue.html
val poolClientFlow = Http().cachedHostConnectionPool[Promise[HttpResponse]]("akka.io")
val queue =
  Source.queue[(HttpRequest, Promise[HttpResponse])](QueueSize, OverflowStrategy.dropNew)
    .via(poolClientFlow)
    .toMat(Sink.foreach({
      case ((Success(resp), p)) => p.success(resp)
      case ((Failure(e), p))    => p.failure(e)
    }))(Keep.left)
    .run()

def queueRequest(request: HttpRequest): Future[HttpResponse] = {
  val responsePromise = Promise[HttpResponse]()
  queue.offer(request -> responsePromise).flatMap {
    case QueueOfferResult.Enqueued    => responsePromise.future
    case QueueOfferResult.Dropped     => Future.failed(new RuntimeException("Queue overflowed. Try again later."))
    case QueueOfferResult.Failure(ex) => Future.failed(ex)
    case QueueOfferResult.QueueClosed => Future.failed(new RuntimeException("Queue was closed (pool shut down) while running the request. Try again later."))
  }
}

val responseFuture: Future[HttpResponse] = queueRequest(HttpRequest(uri = "/"))
```

### 主机层 API + 流式风格

直接使用流式 API 更佳。这几乎消除了中间缓存，因为数据是在请求流入时即时产生的。将请求视为 `Source[(HttpRequest, ...)]` 流，则当该主机的连接有 **足够处理能力** 时，连接池将取出请求进行处理。

```scala
import java.nio.file.Path

import scala.util.{ Failure, Success }
import scala.concurrent.Future

import akka.NotUsed
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import akka.stream.scaladsl._

import akka.http.scaladsl.model.Multipart.FormData
import akka.http.scaladsl.marshalling.Marshal

implicit val system = ActorSystem()
import system.dispatcher // to get an implicit ExecutionContext into scope
implicit val materializer = ActorMaterializer()

case class FileToUpload(name: String, location: Path)

def filesToUpload(): Source[FileToUpload, NotUsed] = ???

val poolClientFlow =
  Http().cachedHostConnectionPool[FileToUpload]("akka.io")

def createUploadRequest(fileToUpload: FileToUpload): Future[(HttpRequest, FileToUpload)] = {
  val bodyPart =
    // fromPath will use FileIO.fromPath to stream the data from the file directly
    FormData.BodyPart.fromPath(fileToUpload.name, ContentTypes.`application/octet-stream`, fileToUpload.location)

  val body = FormData(bodyPart) // only one file per upload
  Marshal(body).to[RequestEntity].map { entity => // use marshalling to create multipart/formdata entity
    // build the request and annotate it with the original metadata
    HttpRequest(method = HttpMethods.POST, uri = "http://example.com/uploader", entity = entity) -> fileToUpload
  }
}

// you need to supply the list of files to upload as a Source[...]
filesToUpload()
  // The stream will "pull out" these requests when capacity is available.
  // When that is the case we create one request concurrently
  // (the pipeline will still allow multiple requests running at the same time)
  .mapAsync(1)(createUploadRequest)
  // then dispatch the request to the connection pool
  .via(poolClientFlow)
  // report each response
  // Note: responses will not come in in the same order as requests. The requests will be run on one of the
  // multiple pooled connections and may thus "overtake" each other.
  .runForeach {
    case (Success(response), fileToUpload) =>
      // TODO: also check for response status code
      println(s"Result for file: $fileToUpload was successful: $response")
      response.discardEntityBytes() // don't forget this
    case (Failure(ex), fileToUpload) =>
      println(s"Uploading file $fileToUpload failed with $ex")
  }
```
