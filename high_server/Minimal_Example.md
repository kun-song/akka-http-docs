# 最简单的例子

如下，是使用路由 DSL 实现的，完整的、非常基础的 Akka HTTP 应用：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model._
import akka.http.scaladsl.server.Directives._
import akka.stream.ActorMaterializer
import scala.io.StdIn

object WebServer {
  def main(args: Array[String]) {

    implicit val system = ActorSystem("my-system")
    implicit val materializer = ActorMaterializer()
    // needed for the future flatMap/onComplete in the end
    implicit val executionContext = system.dispatcher

    val route =
      path("hello") {
        get {
          complete(HttpEntity(ContentTypes.`text/html(UTF-8)`, "<h1>Say hello to akka-http</h1>"))
        }
      }

    val bindingFuture = Http().bindAndHandle(route, "localhost", 8080)

    println(s"Server online at http://localhost:8080/\nPress RETURN to stop...")
    StdIn.readLine() // let it run until user presses return
    bindingFuture
      .flatMap(_.unbind()) // trigger unbinding from the port
      .onComplete(_ => system.terminate()) // and shutdown when done
  }
}
```

该例子在 localhost 上启动 HTTP 服务器，并响应 `/hello` 路径的 GET 请求。

>{% em color="red" %}API 可能变化{% endem %}

>下例使用了实验性特性，且在以后的 Akka HTTP 中，其 API 极有可能变化。更多信息，请查看 Akka 文档中的 [The @DoNotInherit and @ApiMayChange markers](https://doc.akka.io/docs/akka/2.4.19/common/binary-compatibility-rules.html#The_@DoNotInherit_and_@ApiMayChange_markers)。

```scala
import akka.http.scaladsl.model.{ ContentTypes, HttpEntity }
import akka.http.scaladsl.server.HttpApp
import akka.http.scaladsl.server.Route

// Server definition
object WebServer extends HttpApp {
  override def routes: Route =
    path("hello") {
      get {
        complete(HttpEntity(ContentTypes.`text/html(UTF-8)`, "<h1>Say hello to akka-http</h1>"))
      }
    }
}

// Starting the server
WebServer.startServer("localhost", 8080)
```

查看 [HttpApp Bootstrap](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/HttpApp.html) 获取更多此类情动服务器的方法。
