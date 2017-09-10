# HTTP 模型

Akka HTTP 模型完全不可变、基于 `case` 类构建，对主要的 HTTP 数据结构，比如 HTTP 请求、响应、常见（请求、响应）头，Akka HTTP 都有对应模型。

Akka HTTP 模型位于 `akka-http-core`，是大部分 Akka HTTP API 的组成基础。

## 概览

因为 `akka-http-core` 提供了主要 HTTP 数据结构，所以在很多代码中都存在以下引入：

```scala
import akka.http.scaladsl.model._
```

以上将 HTTP 相关的类型引入到当前作用域，主要包括：

* `HttpRequest` 和 `HttpResponse`，这是最核心的消息模型
* `headers`，包含所有预定义的 HTTP 头模型，以及相关支持类型
* `Uri` `HttpMethods` `MediaTypes` `StatusCodes` 等支持类型

一个常见模式是，对于 HTTP 标准定义的不同实体，Akka HTTP 将实体的模型定义为不可变类型（class/trait），而实体的实例定义在 **伴生对象** 中，其名字为类型名加上结尾的 `s`。

例如：

* `HttpMethod` 的实例定义在 `HttpMethods` 对象中
* `HttpCharSet` 的实例定义在 `HttpCharSets` 对象中
* `HttpEncoding` 的实例定义在 `HttpEncodings` 对象中
* `HttpProtocol` 实例定义在 `HttpProtocols` 对象中
* `MediaType` 实例定义在 `MediaTypes` 对象中
* `StatusCode` 实例定义在 `StatusCodes` 对象中

## HttpRequest

`HttpRequest` 和 `HttpResponse` 是表示 HTTP 消息的最基本 `case` 类。

`HttpRequest` 组成如下：

* HTTP 方法（GET, POST ...）
* URI（参考 [URI model]()）
* headers 序列
* 实体（请求体）
* 协议

下面几个例子展示了如何构造 `HttpRequest`：

```scala
import HttpMethods._

// construct a simple GET request to `homeUri`
val homeUri = Uri("/abc")
HttpRequest(GET, uri = homeUri)

// construct simple GET request to "/index" (implicit string to Uri conversion)
HttpRequest(GET, uri = "/index")

// construct simple POST request containing entity
val data = ByteString("abc")
HttpRequest(POST, uri = "/receive", entity = data)

// customize every detail of HTTP request
import HttpProtocols._
import MediaTypes._
import HttpCharsets._
val userData = ByteString("abc")
val authorization = headers.Authorization(BasicHttpCredentials("user", "pass"))
HttpRequest(
  PUT,
  uri = "/user",
  entity = HttpEntity(`text/plain` withCharset `UTF-8`, userData),
  headers = List(authorization),
  protocol = `HTTP/1.0`)
```

`HttpRequest.apply` 的所有参数都有默认值，故诸如 `headers` 等参数，若没有值则无需设置。很多参数类型（比如 `HttpEntity` `Uri`）定义了常见场景下的 {% em %}隐式转换{% endem %}，简化了请求、响应实例的创建。

### 合成请求头

部分场景下，需要与 RFC 标准行为略有差异，比如 Amazon S3 将 URL `path` 部分中的 `+` 字符视为空格，但 RFC 标准实际规定，只有路径的 `query` 部分中的 `+` 才能这样处理。

为了处理此类极端场景（{% em %}edge cases{% endem %}），Akka HTTP 可以通过 {% em %}synthetic headers{% endem %} 向请求提供额外的、非标准信息。这些头没有传递到客户端，而是由 {% em %}请求引擎{% endem %} 处理，用于覆盖默认行为。

例如，为了提供原始的请求 URI，避开默认的 URL 标准化（url normalization），可以如下操作：

```scala
import akka.http.scaladsl.model.headers.`Raw-Request-URI`
val req = HttpRequest(uri = "/ignored", headers = List(`Raw-Request-URI`("/a/b%2Bc")))
```

## HttpResponse

`HttpResponse` 组成部分如下：

* 状态码
* headers 序列
* 实体
* 协议

下面展示了构建 `HttpResponse` 的几种方式：

```scala
import akka.http.scaladsl.model.HttpResponse
import akka.http.scaladsl.model.StatusCodes._
import akka.http.scaladsl.model.headers

// simple OK response without data created using the integer status code
HttpResponse(200)

// 404 response created using the named StatusCode constant
HttpResponse(NotFound)

// 404 response with a body explaining the error
HttpResponse(404, entity = "Unfortunately, the resource couldn't be found.")

// A redirecting response containing an extra header
val locationHeader = headers.Location("http://example.com/other")
HttpResponse(Found, headers = List(locationHeader))
```

至今为止，`HttpEntity` 构造器只接受 `String` 或者 `ByteString`，Akka HTTP 定义了一些 `HttpEntity` 的子类，它们使用字节流创建实体。

## HttpEntity

`HttpEntity` 携带消息内容、消息的 `Content-Type` 以及 `Content-Length`（若可知）。Akka HTTP 定义了 5 种不同 `HttpEntity`，用于表示消息发送、接受的 5 种不同方式。

#### HttpEntity.Strict

最简单的实体，当 **整个实体** 都处于内存中时使用，它包裹 `ByteString`，具备 `Content-Length`（译者注：因为实体全部在内存中，所以长度可知，以未分块形式发送），表示标准的、未分块的 `HttpEntity`。

#### HttpEntity.Default

普通的、未分块的 HTTP/1.1 消息实体，消息的 `Content-Length` 可知，且消息内容为 `Source[ByteString]`（只能被 materialized 一次）。当消息内容的 **实际长度** 与 `Content-Length` 不同时，会产生错误。

`Strict` 和 `Default` 只是 API 上的区分，在网络上，两者表现一致。

#### HttpEntity.Chunked

表示 HTTP/1.1 中的分块内容（携带 `Transfer-Encoding: chunked` 消息头），消息长度不可知，单个块使用 `Source[HttpEntity.ChunkStreamPart]` 表示。`ChunkStreamPart` 要么是非空的 `Chunk`，要么是 `LastChunk`（包含可选的 trailer headers）。

整个流包含一个或多个 `Chunk`，使用可选的 `LastChunk` 终止。

> 译者注：分块传输编码（Chunked transfer encoding）是 HTTP 中的一种数据传输机制，允许服务器发送给客户端的数据可以分成多个部分，多次传输完成，更多内容，查看 [RFC chunked content](https://tools.ietf.org/html/rfc7230#section-4.1)。

#### HttpEntity.CloseDelimited

未分块、长度未知、需要通过关闭连接（`Connection: close`）确定结束边界的实体，内容用 `Source[ByteString]` 表示。因为只能通过发送该类型的是实体来关闭连接，所以只能用于 {% em %}服务端{% endem %} 发送响应。

`CloseDelimited` 实体主要用于与 HTTP/1.0 兼容，因为 HTTP/1.0 不支持分块传输编码（chunked transfer encoding）。如果不需要与老旧代码兼容，则不要使用 `CloseDelimited` 实体，因为隐式的通过关闭连接来结束响应（terminate-by-connection-close）健壮性较差，尤其有代理存在时。并且，该类型实体不能复用（TCP）连接，会严重降低性能。

尽量使用 `HttpEntity.Chunked`！

#### HttpEntity.IndefiniteLength

长度未知的流式实体，用于 `Multipart.BodyPart`。

`Strict` `Default` `Chunked` 实体皆为 `HttpEntity.Regular` 的子类，故可用于请求和响应。而 `CloseDelimited` 只能用于响应。

流式实体类型（除 `Strict` 以外的所有）不能被共享，也不能被序列化。要想创建实体或消息的 `strict`、可共享副本，可以使用 `HttpEntity.toStrict` 或 `HttpMessage.toStrict` 方法，该方法返回一个 `Future`，原本的实体内容被收集到一个 `ByteString` 中。

`HttpEntity` 伴生对象包含使用常见类型创建实体的辅助函数。

使用模式匹配，可以对 `HttpEntity` 每个子类提供特殊处理。然而大多数场景下，`HttpEntity` 的接收者并不关心具体的 `HttpEntity` 类型，也不关心数据是如何传输的，因此 Akka HTTP 提供了 `HttpEntity.dataBytes` 方法，返回 `Source[ByteString, Any]`，用于获取 {% em %}任意类型{% endem %} `HttpEntity` 的实体数据。

##### 如何选择 `HttpEntity` 子类型？

* 数据量很小，且已经在内存中时，使用 `Strict`；
* 数据由流式数据产生，且长度未知时，使用 `Default`；
* 长度未知时，使用 `Chunked`；
* 若客户端不支持分块传输编码时，使用 `CloseDelimited` 作为响应；其他场景都使用 `Chunked`！
* 用于 `Multipart.BodyPart`，内容长度未知时，使用 `IndefiniteLength`；

##### 注意

当接收到非 `Strict` 类型的消息（非 `Strict` 类型的其他 `HttpEntity` 都是流式数据）时，只有通过消费实体流，才能继续、请求获取其余数据；若不消费当前的实体流，则连接将保持 `stalled`，当前消息的实体会 {% em %}阻塞{% endem %} 流，故无法读取后续的请求、响应。

所以即使对实体内容不感兴趣，也需要消费实体。

### 限制消息实体长度

Akka HTTP 从网络获取的所有消息实体，都会自动进行长度校验。该校验保证实体长度小于或等于 `max-content-length` 的值，这是防止 DOS（denial-of-service）攻击的重要手段。然而，若对所有请求（响应）施加同样的长度限制，有时过于僵化，比如有的应用希望对某些请求（响应）施加很高的限制长度，而其余请求（响应）则遵循全局限制。

为了增加灵活性，按需定制实体的大小限制，`HttpEntity` 的 `withSizeLimit` 方法，可以针对特定实体，为其调整全局配置的长度限制。这意味着应用可以从 HTTP 层接收所有请求（或响应），即使其实体的 `Content-Length` 超过全局配置的最大长度（因为可能修改了实体的最大长度）。

只有当实体中的真正数据流 `Source` 物质化以后，长度校验才会真正执行。若校验失败，则对应的流将被终止，并抛出 `EntityStreamSizeException`（若 `Content-Length` 已知，则在物质化时抛出，否则在读取超过限制的数据时抛出）。

在 `Strict` 实体上调用 `withSizeLimit` 方法时，若实体的 `Content-Length` 在边界以内，则返回该实体本身；否则返回只有一个元素数据源的 `Default` 实体。这允许之后（数据源被物质化之前）对实体最大长度进行修正（refinement）。

从 HTTP 层产生的消息实体默认都会携带其最大长度，该数值在 `max-content-length` 中配置。若实体被转化后，`Content-Length` 发生变化，同时施加了新的最大长度，则使用新 `Content-Length` 与新长度限制进行长度校验；若实体被转化后，`Content-Length` 发生变化，但未施加新的最大长度，则使用原来的 `Content-Length` 与原来的最大长度进行长度校验。在这两种场景下，各自的校验方式都是合乎直觉的。

### 特殊处理 HEAD 请求

[RFC 7230](https://tools.ietf.org/html/rfc7230#section-3.3.3) 对 HTTP 消息的实体长度有明确定义。

下面的规则要求 Akka HTTP 对 HEAD 请求做特殊处理：

>Any response to a HEAD request and any response with a 1xx (Informational), 204 (No Content), or 304 (Not Modified) status code is always terminated by the first empty line after the header fields, regardless of the header fields present in the message, and thus cannot contain a message body.

对 HEAD 请求的响应引入一定的复杂处理，该响应的 `Content-Length` 或 `Transfer-Encoding` 头都可以出现，但此时实体为空。Akka HTTP 通过允许在 HEAD 响应中使用 `HttpEntity.Default` 和 `HttpEntity.Chunked`，同时允许数据流为空，来模拟这种场景。

并且，当 HEAD 响应包含 `HttpEntity.CloseDelimited` 实体时，当响应发送结束后，Akka HTTP **不会关闭连接**，从而允许无 `Content-Length` 头的 HEAD 响应可以通过持久 HTTP 连接发送。

## Header 模型

对常见的 HTTP 头部，Akka HTTP 都有建模。头部的解析、展示全部自动完成，应用无需接入。对于 Akka HTTP 没有建模的头部，使用 `RawHeader` 表示（简单的）。

下面例子展示了如何处理头部：

```scala
// create a ``Location`` header
val loc = Location("http://example.com/other")

// create an ``Authorization`` header with HTTP Basic authentication data
val auth = Authorization(BasicHttpCredentials("joe", "josepp"))

// custom type
case class User(name: String, pass: String)

// a method that extracts basic HTTP credentials from a request
def credentialsOfRequest(req: HttpRequest): Option[User] =
  for {
    Authorization(BasicHttpCredentials(user, pass)) <- req.header[Authorization]
  } yield User(user, pass)
```

## HTTP Headers

当 Akka HTTP 服务端接受到 HTTP 请求时，会尝试解析全部 HTTP 头部到各自对应的模型类。无论解析是否成功，HTTP 层会将全部接收到的头部传递给应用。{% em %}未知头部{% endem %}、存在无效语法（header parser 检测）的头部，将解析为 `RawHeader` 实例。对于解析错误，将根据 `illegal-header-warnings` 配置决定是否在日志中打印警告。

部分头部在 HTTP 中 {% em %}地位特别{% endem %}，因此处理方式与普通头部不同：

#### Content-Type

HTTP 消息中的 `Content-Type` 被建模为 `HttpEntity` 中的 `contentType` 字段，所以 `Content-Type` 头在消息的 `headers` 序列中并不存在。

显式添加到请求、响应的 `headers` 序列中的 `Content-Type` 头部实例在网络传输时将被忽略，并会在日志中触发警告！

#### Transfer-Encoding

Akka HTTP 使用 `HttpEntity.Chunked` 实体表示具有 `Transfer-Encoding: chunked` 头部的消息。因此若分块消息（chunked messages）中没有嵌套的 transfer encoding，则其 `headers` 序列中无 `Transfer-Encoding` 头部。

实现添加到请求、响应的 `headers` 序列中的 `Transfer-Encoding` 头部实例在网络传输时将被忽略，并会在日志中触发警告。

#### Content-Length

消息的内容长度使用 `HttpEntity` 表示，因此消息的 `headers` 序列中不存在 `Content-Length` 头部。

实现添加到请求、响应的 `headers` 序列中的 `Content-Length` 头部实例在网络传输时将被忽略，并会在日志中触发警告。

#### Server

`Server` 头部会自动添加到所有 HTTP 响应中，其值可以通过 `akka.http.server.server-header` 进行配置。应用可以将其显示添加到响应 `headers` 序列中，这将覆盖配置文件中的值。

#### User-Agent

`User-Agent` 头部会自动添加到所有 HTTP 请求中，其值可以通过 `akka.http.client.user-agent-header` 进行配置。应用可以将其显示添加到请求 `headers` 序列中，这将覆盖配置文件中的值。

#### Date

`Date` 头部会自动添加到 HTTP 响应中，但可手动指定，从而覆盖默认值。

#### Connection

在服务端，Akka HTTP 会检测显式添加的 `Connection: close` 响应头部，并尊重应用希望（发送完本次响应后关闭）关闭连接的意愿。但决定是否真正关闭的逻辑比较复杂，会综合考虑请求的方法、协议、可能的 `Connection` 头部，以及响应的协议、实体、可能的 `Connection` 头部等因素。查看 [test this](https://github.com/akka/akka-http/tree/v10.0.10/akka-http-core/src/test/scala/akka/http/impl/engine/rendering/ResponseRendererSpec.scala) 获取详细信息。

#### Strict-Transport-Security

HTTP Strict Transport Security(HSTS) 是一种网络安全机制，该机制依赖 `Strict-Transport-Security` 头部进行客户端、服务端通信。HSTS 可以防止 SSL-stripping 中间人攻击，该类攻击将 HTTPS 请求替换为 HTTP 请求，而 HSTS 可以通知浏览器，本站点必须通过 TLS/SSL 连接，从而确保安全。

## 自定义 Headers

有时需要自定义非 HTTP 标准的头部，并享受如同内置类型一样的便利。

因为大家对头部的操作方式很多样（比如，使用 `CustomHeader` 对 `RawHeader` 进行模式匹配 ...），所以 Akka HTTP 提供了自定义头部类型，以及其伴生类的辅助特质：

```scala
final class ApiTokenHeader(token: String) extends ModeledCustomHeader[ApiTokenHeader] {

  override def companion: ModeledCustomHeaderCompanion[ApiTokenHeader] = ApiTokenHeader

  override def value(): String = token

  override def renderInRequests(): Boolean = false

  override def renderInResponses(): Boolean = false
}

object ApiTokenHeader extends ModeledCustomHeaderCompanion[ApiTokenHeader] {

  override def name: String = "apiKey"

  override def parse(value: String): Try[ApiTokenHeader] = Try(new ApiTokenHeader(value))
}
```

上面定义的头部，可以在如下场景中使用：

```scala
val ApiTokenHeader(t1) = ApiTokenHeader("token")
t1 should ===("token")

val RawHeader(k2, v2) = ApiTokenHeader("token")
k2 should ===("apiKey")
v2 should ===("token")

// will match, header keys are case insensitive
val ApiTokenHeader(v3) = RawHeader("APIKEY", "token")
v3 should ===("token")

intercept[MatchError] {
  // won't match, different header name
  val ApiTokenHeader(v4) = DifferentHeader("token")
}

intercept[MatchError] {
  // won't match, different header name
  val RawHeader("something", v5) = DifferentHeader("token")
}

intercept[MatchError] {
  // won't match, different header name
  val ApiTokenHeader(v6) = RawHeader("different", "token")
}
```

在头部指令内部使用的例子在 [headerValuePF](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/directives/header-directives/headerValuePF.html) 中，如下所示：

```scala
def extractFromCustomHeader = headerValuePF {
  case t @ ApiTokenHeader(token) ⇒ s"extracted> $t"
  case raw: RawHeader            ⇒ s"raw> $raw"
}

val routes = extractFromCustomHeader { s ⇒
  complete(s)
}

Get().withHeaders(RawHeader("apiKey", "TheKey")) ~> routes ~> check {
  status should ===(StatusCodes.OK)
  responseAs[String] should ===("extracted> apiKey: TheKey")
}

Get().withHeaders(RawHeader("somethingElse", "TheKey")) ~> routes ~> check {
  status should ===(StatusCodes.OK)
  responseAs[String] should ===("raw> somethingElse: TheKey")
}

Get().withHeaders(ApiTokenHeader("TheKey")) ~> routes ~> check {
  status should ===(StatusCodes.OK)
  responseAs[String] should ===("extracted> apiKey: TheKey")
}
```

也可以直接继承 `CustomHeader`，样板代码更少，缺点是与 `RawHeader` 的模式匹配不是开箱即用的，限制了其在 Akka HTTP 路由层的使用。

>注意：
自定义头部时，优先继承 `ModeledCustomHeader`，而非 `CustomHeader`，因为可以自动获得所有内置类型都具备的模式匹配能力（比如自定义头部与 `RawHeader` 的模式匹配，这在路由层很常见）。

## Parsing/Rendering

HTTP 数据结构的解析、渲染对大多数类型已经高度优化，目前没有提供公共 API 用于解析（或渲染为）字符串、字节数组。

## 注册自定义的 Media Types

Akka HTTP [预定义](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/model/MediaTypes$.html)了大多数常见的媒体类型，并在解析 HTTP 消息时生成它们。有时可能需要自定义一种媒体类型，并通知解析器该如何处理该类型，比如告诉解析器 `application/custom` 应该视为具备 `WithFixedCharset` 的 `NonBinary`。这需要在服务端的 `ParserSetting` 配置中注册该媒体类型：

```scala
// similarily in Java: `akka.http.javadsl.settings.[...]`
import akka.http.scaladsl.settings.ParserSettings
import akka.http.scaladsl.settings.ServerSettings

// define custom media type:
val utf8 = HttpCharsets.`UTF-8`
val `application/custom`: WithFixedCharset =
  MediaType.customWithFixedCharset("application", "custom", utf8)

// add custom media type to parser settings:
val parserSettings = ParserSettings(system).withCustomMediaTypes(`application/custom`)
val serverSettings = ServerSettings(system).withParserSettings(parserSettings)

val routes = extractRequest { r ⇒
  complete(r.entity.contentType.toString + " = " + r.entity.contentType.getClass)
}
val binding = Http().bindAndHandle(routes, host, port, settings = serverSettings)
```

参考媒体类型的 [注册树](https://en.wikipedia.org/wiki/Media_type#Registration_trees)，以便将该类型置于合理的位置。

## 注册自定义的 Status Codes

与媒体类型类似，Akka HTTP [预定义](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/model/StatusCodes$.html)了常见的状态码，但有时需要自定义的状态码（或被强制使用返回自定义状态码的 API）。与注册媒体类型相同，通过配置 `ParserSettings` 来注册自定义的状态码。

```scala
// similarily in Java: `akka.http.javadsl.settings.[...]`
import akka.http.scaladsl.settings.{ ParserSettings, ServerSettings }

// define custom status code:
val LeetCode = StatusCodes.custom(777, "LeetCode", "Some reason", isSuccess = true, allowsEntity = false)

// add custom method to parser settings:
val parserSettings = ParserSettings(system).withCustomStatusCodes(LeetCode)
val serverSettings = ServerSettings(system).withParserSettings(parserSettings)

val clientConSettings = ClientConnectionSettings(system).withParserSettings(parserSettings)
val clientSettings = ConnectionPoolSettings(system).withConnectionSettings(clientConSettings)

val routes =
  complete(HttpResponse(status = LeetCode))

// use serverSettings in server:
val binding = Http().bindAndHandle(routes, host, port, settings = serverSettings)

// use clientSettings in client:
val request = HttpRequest(uri = s"http://$host:$port/")
val response = Http().singleRequest(request, settings = clientSettings)

// futureValue is a ScalaTest helper:
response.futureValue.status should ===(LeetCode)
```

## 注册自定义的 HTTP 方法

在[预定义](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/model/HttpMethods$.html)的 HTTP 方法以外，Akka HTTP 允许自定义 HTTP 方法。要自定义 HTTP 方法，首先要定义该方法，然后将其添加到解析器配置中：

```scala
import akka.http.scaladsl.settings.{ ParserSettings, ServerSettings }

// define custom method type:
val BOLT = HttpMethod.custom("BOLT", safe = false,
  idempotent = true, requestEntityAcceptance = Expected)

// add custom method to parser settings:
val parserSettings = ParserSettings(system).withCustomMethods(BOLT)
val serverSettings = ServerSettings(system).withParserSettings(parserSettings)

val routes = extractMethod { method ⇒
  complete(s"This is a ${method.name} method request.")
}
val binding = Http().bindAndHandle(routes, host, port, settings = serverSettings)

val request = HttpRequest(BOLT, s"http://$host:$port/", protocol = `HTTP/1.1`)
```
