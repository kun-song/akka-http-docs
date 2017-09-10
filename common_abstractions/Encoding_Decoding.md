# 编码、解码

[Http 标准](http://tools.ietf.org/html/rfc7231#section-3.1.2.1) 定义了 `Content-Encoding` 头部，用以表明 HTTP 消息的实体是否被 “编码”，以及以何种算法进行编码。唯一常用的内容编码为压缩算法。

目前，Akka HTTP 支持使用 `gzip` 和 `deflate` 编码对 HTTP 请求和响应进行压缩、解压，其核心逻辑在 [akka.http.scaladsl.coding](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/coding/index.html) 包中。

## 服务端

在服务端，内容编码、解码默认不开启，必须显示请求，要使用 [路由 DSL](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/index.html) 支持内容编码，请查阅 [CodingDirectives](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/coding-directives/index.html)。

## 客户端

在客户端，目前没有对响应进行编码的高层的、自动的支持。

下面的例子展示了如何基于 `Content-Encoding` 头部对响应手动解码：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.coding.{ Gzip, Deflate, NoCoding }
import akka.http.scaladsl.model._, headers.HttpEncodings
import akka.stream.ActorMaterializer

import scala.concurrent.Future

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
import system.dispatcher

val http = Http()

val requests: Seq[HttpRequest] = Seq(
  "https://httpbin.org/gzip", // Content-Encoding: gzip in response
  "https://httpbin.org/deflate", // Content-Encoding: deflate in response
  "https://httpbin.org/get" // no Content-Encoding in response
).map(uri ⇒ HttpRequest(uri = uri))

def decodeResponse(response: HttpResponse): HttpResponse = {
  val decoder = response.encoding match {
    case HttpEncodings.gzip ⇒
      Gzip
    case HttpEncodings.deflate ⇒
      Deflate
    case HttpEncodings.identity ⇒
      NoCoding
  }

  decoder.decodeMessage(response)
}

val futureResponses: Future[Seq[HttpResponse]] =
  Future.traverse(requests)(http.singleRequest(_).map(decodeResponse))

futureResponses.futureValue.foreach { resp =>
  system.log.info(s"response is ${resp.toStrict(1.second).futureValue}")
}

system.terminate()
```
