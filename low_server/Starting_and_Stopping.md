# 启动与停止

启动 Akka HTTP 服务器最简单的方式为，调用 [akka.http.scaladsl.Http](https://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/Http$.html) 的 `bind` 方法：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.stream.ActorMaterializer
import akka.stream.scaladsl._

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()
implicit val executionContext = system.dispatcher

val serverSource: Source[Http.IncomingConnection, Future[Http.ServerBinding]] =
  Http().bind(interface = "localhost", port = 8080)
val bindingFuture: Future[Http.ServerBinding] =
  serverSource.to(Sink.foreach { connection => // foreach materializes the source
    println("Accepted new connection from " + connection.remoteAddress)
    // ... and then actually handle the connection
  }).run()
```

`Http().bind` 方法的参数为要绑定的接口和端口（interface and port），并注册接收进入的 HTTP 连接。另外，该方法同样允许定义套接字选项，以及更多服务器配置。

`bind` 方法返回值为 `Source[Http.IncomingConnection]`，应用必须消耗该 `Source` 以接收进入的请求。在 `Source` 物质化后才会真正进行绑定。若绑定失败（比如端口已被占用），则会立即停止已物质化的流，并抛出异常。当进入连接源的订阅者取消订阅时，释放绑定（释放底层的套接字）。也可以使用 `Http.ServerBinding` 的 `unbind` 方法。通过 `Http.ServerBinding` 可以获取套接字的真实地址，当绑定端口为 0 时（OS 自动选取空闲端口）非常有用。
