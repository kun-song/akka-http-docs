# HTTP 客户端 API

客户端 API 提供请求 HTTP 服务器的方法，客户端与服务端共享相同的 `HttpRequest` 和 `HttpResponse` 抽象，但添加了 **连接池** 的概念，通过允许多个请求（它们请求相同的服务端）复用与服务端的 TCP 连接，可以提升性能。

```
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

查阅 [消费基于 HTTP 的服务](http://doc.akka.io/docs/akka-http/current/scala/http/client-side/index.html) 获取更多细节。
