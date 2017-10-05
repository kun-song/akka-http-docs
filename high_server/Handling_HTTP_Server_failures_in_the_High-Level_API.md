# 处理服务器错误

在初始化或运行 Akka HTTP 服务器时，很多场景都可能发生错误。默认情况下，Akka 将打印这些错误，有时除简单打印外，还需要其他措施，比如关闭 Actor 系统，或通知外部监控等。

## 绑定错误

第一种错误类型是，服务器无法绑定到指定端口，例如当该端口已被占用，或无该端口的权限（比如只有 root 用户可用）。该场景下 *binding future* 将立刻失败，可以通过监听该 `Future` 的完成事件来做出反应：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.Http.ServerBinding
import akka.http.scaladsl.server.Directives._
import akka.stream.ActorMaterializer

import scala.concurrent.Future

object WebServer {
  def main(args: Array[String]) {
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    // needed for the future foreach in the end
    implicit val executionContext = system.dispatcher

    val handler = get {
      complete("Hello world!")
    }

    // let's say the OS won't allow us to bind to 80.
    val (host, port) = ("localhost", 80)
    val bindingFuture: Future[ServerBinding] =
      Http().bindAndHandle(handler, host, port)

    bindingFuture.failed.foreach { ex =>
      log.error(ex, "Failed to bind to {}:{}!", host, port)
    }
  }
}
```

>注意

>若要从更底层视角了解各种失败场景，参考 [底层 API 中如何处理服务器失败](../low_server/Handling_HTTP_Server_failures_in_the_Low-Level_API.md)。

## 路由 DSL 中的错误与异常

路由 DSL 中发生的错误、异常由 `ExceptionHandler` 处理，更多细节参考 [Exception Handling](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/exception-handling.html)。`ExceptionHandler` 将异常转化为 `HttpResponse`，其中包含错误码和错误描述信息。

## 文件上传

[FileUploadDirectives](https://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/directives/file-upload-directives/index.html) 介绍支持文件上传的指令。

通过接受 `Multipart.FormData` 实体，可以实现浏览器表单文件上传，因为消息体、消息体 payload 是 `Source`，并非立即可得，所以需要消费它们的流。

下面是一个简单例子，将接受到的文件写入磁盘的临时文件中，将表单字段写入虚拟的数据库：

```scala
val uploadVideo =
  path("video") {
    entity(as[Multipart.FormData]) { formData =>

      // collect all parts of the multipart as it arrives into a map
      val allPartsF: Future[Map[String, Any]] = formData.parts.mapAsync[(String, Any)](1) {

        case b: BodyPart if b.name == "file" =>
          // stream into a file as the chunks of it arrives and return a future
          // file to where it got stored
          val file = File.createTempFile("upload", "tmp")
          b.entity.dataBytes.runWith(FileIO.toPath(file.toPath)).map(_ =>
            (b.name -> file))

        case b: BodyPart =>
          // collect form field values
          b.toStrict(2.seconds).map(strict =>
            (b.name -> strict.entity.data.utf8String))

      }.runFold(Map.empty[String, Any])((map, tuple) => map + tuple)

      val done = allPartsF.map { allParts =>
        // You would have some better validation/unmarshalling here
        db.create(Video(
          file = allParts("file").asInstanceOf[File],
          title = allParts("title").asInstanceOf[String],
          author = allParts("author").asInstanceOf[String]))
      }

      // when processing have finished create a response for the user
      onSuccess(allPartsF) { allParts =>
        complete {
          "ok!"
        }
      }
    }
  }
```

除将文件写入临时文件，还可以在文件到来时对其进行处理。下例接受 `.csv` 文件，将其解析成行，然后将其发送到 Actor 中进一步处理：

```scala
val splitLines = Framing.delimiter(ByteString("\n"), 256)

val csvUploads =
  path("metadata" / LongNumber) { id =>
    entity(as[Multipart.FormData]) { formData =>
      val done: Future[Done] = formData.parts.mapAsync(1) {
        case b: BodyPart if b.filename.exists(_.endsWith(".csv")) =>
          b.entity.dataBytes
            .via(splitLines)
            .map(_.utf8String.split(",").toVector)
            .runForeach(csv =>
              metadataActor ! MetadataActor.Entry(id, csv))
        case _ => Future.successful(Done)
      }.runWith(Sink.ignore)

      // when processing have finished create a response for the user
      onSuccess(done) { _ =>
        complete {
          "ok!"
        }
      }
    }
  }
```
