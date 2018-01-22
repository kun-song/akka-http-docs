# FileUploadDirectives

* uploadedFile
* fileUpload

## uploadedFile

### 签名

```scala
def uploadedFile(fieldName: String): Directive1[(FileInfo, File)]
```

### 描述

将以 `multipart` 表单形式上传的文件内容导入磁盘上的 **临时文件**，并以提取值形式提供关于该次上传的元数据和 `File`。

若写入磁盘时发生错误，则请求失败，并返回对应异常；若表单中没有指定名字的字段，则拒绝该请求；若表单中有多个相同名字的字段，则将处理第一个，并忽略剩余。

>注意
>`uploadedFile` 指令会把请求内容导入文件，但必须等待 **全部写入** 后，才能开始处理文件内容；对于流式 API，更倾向于使用 `fileUpload` 指令，它允许对进入的数据字节进行流式处理（即不需要等待所有内容成功写入磁盘）。

### 示例

```scala
import akka.http.scaladsl.model.{ContentTypes, HttpEntity, Multipart, StatusCodes}
import akka.http.scaladsl.server.Directives.{complete, uploadedFile}
import akka.http.scaladsl.server.Route
import akka.http.scaladsl.testkit.ScalatestRouteTest
import org.scalatest.{Matchers, WordSpec}

class UploadedFileSpec extends WordSpec with Matchers with ScalatestRouteTest {
  val route: Route =
    uploadedFile("csv") {
      case (meta, file) ⇒
        println(file.getPath)  // 打印临时文件的路径
        file.delete()
        complete(StatusCodes.OK)
    }

  val multipartForm =
    Multipart.FormData(
      Multipart.FormData.BodyPart.Strict(
        "csv",
        HttpEntity(ContentTypes.`text/plain(UTF-8)`, "Hello World!"),
        Map("filename" → "data.csv")))

  Post("/", multipartForm) ~> route ~> check {
    status shouldEqual StatusCodes.OK
  }
}
```

## fileUpload

### 签名

```scala
def fileUpload(fieldName: String): Directive1[(FileInfo, Source[ByteString, Any])]
```

### 描述

以流的方式访问以 `multipart` 表单形式上传的文件内容，且以提取值的形式提供上传文件的元数据。

若表单中没有指定名字的字段，则拒绝请求；若有多个相同名字的字段，则仅处理第一个，并忽略其他。

### 示例

```scala
// adding integers as a service ;)
val route =
  extractRequestContext { ctx =>
    implicit val materializer = ctx.materializer
    implicit val ec = ctx.executionContext

    fileUpload("csv") {
      case (metadata, byteSource) =>

        val sumF: Future[Int] =
          // sum the numbers as they arrive so that we can
          // accept any size of file
          byteSource.via(Framing.delimiter(ByteString("\n"), 1024))
            .mapConcat(_.utf8String.split(",").toVector)
            .map(_.toInt)
            .runFold(0) { (acc, n) => acc + n }

        onSuccess(sumF) { sum => complete(s"Sum: $sum") }
    }
  }

// tests:
val multipartForm =
  Multipart.FormData(Multipart.FormData.BodyPart.Strict(
    "csv",
    HttpEntity(ContentTypes.`text/plain(UTF-8)`, "2,3,5\n7,11,13,17,23\n29,31,37\n"),
    Map("filename" -> "primes.csv")))

Post("/", multipartForm) ~> route ~> check {
  status shouldEqual StatusCodes.OK
  responseAs[String] shouldEqual "Sum: 178"
}
```
