# 路由 DSL 概览

[底层服务端 API](./low_server/Introduction.md) 提供了基于 `Flow` 或基于 `Function`，来处理 HTTP 请求的方式，它们简单地将请求映射到响应上：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model.HttpMethods._
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import scala.io.StdIn

object WebServer {

  def main(args: Array[String]) {
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    // needed for the future map/flatmap in the end
    implicit val executionContext = system.dispatcher

    val requestHandler: HttpRequest => HttpResponse = {
      case HttpRequest(GET, Uri.Path("/"), _, _, _) =>
        HttpResponse(entity = HttpEntity(
          ContentTypes.`text/html(UTF-8)`,
          "<html><body>Hello world!</body></html>"))

      case HttpRequest(GET, Uri.Path("/ping"), _, _, _) =>
        HttpResponse(entity = "PONG!")

      case HttpRequest(GET, Uri.Path("/crash"), _, _, _) =>
        sys.error("BOOM!")

      case r: HttpRequest =>
        r.discardEntityBytes() // important to drain incoming HTTP Entity stream
        HttpResponse(404, entity = "Unknown resource!")
    }

    val bindingFuture = Http().bindAndHandleSync(requestHandler, "localhost", 8080)
    println(s"Server online at http://localhost:8080/\nPress RETURN to stop...")
    StdIn.readLine() // let it run until user presses return
    bindingFuture
      .flatMap(_.unbind()) // trigger unbinding from the port
      .onComplete(_ => system.terminate()) // and shutdown when done

  }
}
```

通过对 `HttpRequest` 进行模式匹配实现 REST API 虽然很好；但当服务变大时，就略嫌笨重了，而且不符合 [DRY](http://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 原则。

Akka HTTP 提供了一套灵活的 DSL，用一些可组合的元素（[Directives](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/directives/index.html)）来表示服务接口，更加精确、可读。`Directive` 组合在一起，形成 `Route` 结构，`Route` 可以转化为 *handler `Flow`*（或者 *async handler function*），从而可以直接传递给 `bind` 函数。从 `Route` 到 `Flow` 的转化，可以通过 `Route.handlerFlow` 显式完成，也可以通过 `RouteResult.route2HandlerFlow` 隐式完成。

>注意

>To be picked up automatically, the implicit conversion needs to be provided in the companion object of the source type. However, as Route is just a type alias for RequestContext => Future[RouteResult], there’s no companion object for Route. Fortunately, the [implicit scope](xxx) for finding an implicit conversion also includes all types that are “associated with any part” of the source type which in this case means that the implicit conversion will also be picked up from RouteResult.route2HandlerFlow automatically.

上面的例子用路由 DSL 重新实现如下：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model.{ContentTypes, HttpEntity}
import akka.http.scaladsl.server.Directives._
import akka.stream.ActorMaterializer
import scala.io.StdIn

object WebServer {
  def main(args: Array[String]) {
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    // needed for the future flatMap/onComplete in the end
    implicit val executionContext = system.dispatcher

    val route =
      get {
        pathSingleSlash {
          complete(HttpEntity(ContentTypes.`text/html(UTF-8)`,"<html><body>Hello world!</body></html>"))
        } ~
          path("ping") {
            complete("PONG!")
          } ~
          path("crash") {
            sys.error("BOOM!")
          }
      }

    // `route` will be implicitly converted to `Flow` using `RouteResult.route2HandlerFlow`
    val bindingFuture = Http().bindAndHandle(route, "localhost", 8080)
    println(s"Server online at http://localhost:8080/\nPress RETURN to stop...")
    StdIn.readLine() // let it run until user presses return
    bindingFuture
      .flatMap(_.unbind()) // trigger unbinding from the port
      .onComplete(_ => system.terminate()) // and shutdown when done
  }
}
```

使用如下语句导入路由 DSL 的核心依赖：

```scala
import akka.http.scaladsl.server.Directives._
```

该例子还依赖预定义的对 Scala XML 的支持：

```scala
import akka.http.scaladsl.marshallers.xml.ScalaXmlSupport._
```

上例不足以展示路由 DSL 带来的精确性、可读性，以及代码量的减少，[Long Example](./high_server/Longer_Example.md) 可能更有说服力。

要使用路由 DSL，首先要理解 [Route](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/routes.html) 的概念。
