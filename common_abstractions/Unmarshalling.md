# 反序列化

Unmarshalling 是将底层表示形式（常常为 wire format）转化为高层结构的过程，也称之为 Deserialization 或 Unpicking。

在 Akka HTTP 中，Unmarshalling 是将底层对象，比如 `MessageEntity`（组成 HTTP 请求或响应的 entity body）或 `HttpRequest` 或 `HttpResponse` 转化为类型 `T` 实例的过程。

## 基本设计

从类型 `A` 的实例到类型 `B` 的实例的反序列化由 `Unmarshaller[A, B]` 完成。Akka HTTP 为常用的 Unmarshaller 定义了别名：

```
type FromEntityUnmarshaller[T] = Unmarshaller[HttpEntity, T]
type FromMessageUnmarshaller[T] = Unmarshaller[HttpMessage, T]
type FromResponseUnmarshaller[T] = Unmarshaller[HttpResponse, T]
type FromRequestUnmarshaller[T] = Unmarshaller[HttpRequest, T]
type FromByteStringUnmarshaller[T] = Unmarshaller[ByteString, T]
type FromStringUnmarshaller[T] = Unmarshaller[String, T]
type FromStrictFormFieldUnmarshaller[T] = Unmarshaller[StrictForm.Field, T]
```

其中核心 `Unmarshaller[A, B]` 非常类似于函数 `A => Future[B]`，相对 `Marshaller[A, B]` 要简单一些。反序列化不需要支持内容协商，因此省去了序列化需要的额外两层封装（`List[Marshalling[B]]`）。

## 预定义的 Unmarshaller

Akka HTTP 为大多数常见类型预定义了 Unmarshaller：

* [PredefinedFromStringUnmarshallers](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/unmarshalling/PredefinedFromStringUnmarshallers.html)
  + `Byte`
  + `Short`
  + `Int`
  + `Long`
  + `Float`
  + `Double`
  + `Boolean`
* [PredefinedFromEntityUnmarshallers](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/unmarshalling/PredefinedFromEntityUnmarshallers.html)
  + `Array[Byte]`
  + `ByteString`
  + `Array[Char]`
  + `String`
  + `akka.http.scaladsl.model.FormData`
* [GenericUnmarshallers](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/unmarshalling/GenericUnmarshallers.html)
  + `Unmarshaller[T, T]`
  + `Unmarshaller[Option[A], B]`，若 `Unmarshaller[A, B]` 有效
  + `Unmarshaller[A, Option[B]]`，若 `Unmarshaller[A, B]` 有效

## 隐式转换

Akka HTTP 的 unmarshalling 设施基于 'type class' 方式，即从类型 `A` 到类型 `B` 的转化的 `Unmarshaller` 必须为隐式函数。

大多数 Akka HTTP 预定义的 Unmarshallers 由 `Unmarshaller` 特质的伴生对象提供，即它们随时可用，不需要显式导入。通过在本地作用域自定义 Unmarshaller 可以覆盖默认版本。

## 自定义 Unmarshaller

Akka HTTP 提供了一些便利工具，用于构建用于特定类型的 Unmarshaller。自定义 Unmarshaller 一般不需要直接手动实现 `Unmarshaller` 特质，可以使用 `Unmarshaller` 伴生对象中的构建辅助函数：

```scalads
/**
 * Creates an `Unmarshaller` from the given function.
 */
def apply[A, B](f: ExecutionContext ⇒ A ⇒ Future[B]): Unmarshaller[A, B] =
  withMaterializer(ec => _ => f(ec))

def withMaterializer[A, B](f: ExecutionContext ⇒ Materializer => A ⇒ Future[B]): Unmarshaller[A, B] =
  new Unmarshaller[A, B] {
    def apply(a: A)(implicit ec: ExecutionContext, materializer: Materializer) =
      try f(ec)(materializer)(a)
      catch { case NonFatal(e) ⇒ FastFuture.failed(e) }
  }

/**
 * Helper for creating a synchronous `Unmarshaller` from the given function.
 */
def strict[A, B](f: A ⇒ B): Unmarshaller[A, B] = Unmarshaller(_ => a ⇒ FastFuture.successful(f(a)))

/**
 * Helper for creating a "super-unmarshaller" from a sequence of "sub-unmarshallers", which are tried
 * in the given order. The first successful unmarshalling of a "sub-unmarshallers" is the one produced by the
 * "super-unmarshaller".
 */
def firstOf[A, B](unmarshallers: Unmarshaller[A, B]*): Unmarshaller[A, B] = //...
```

## 继承 Unmarshaller

利用已存在的 Marshaller 来构建自己的 Marshaller 会节省时间，方法是封装已存在的 Marshaller，并将其结果 “重定向” 到自己的类型。

很多时候，你可能需要将已存在的 unmarshallers 的结果转换为自己想要的类型，对此类转换，Akka HTTP 提供了如下方法：

* `baseUnmarshaller.transform`
* `baseUnmarshaller.map`
* `baseUnmarshaller.mapWithInput`
* `baseUnmarshaller.flatMap`
* `baseUnmarshaller.flatMapWithInput`
* `baseUnmarshaller.recover`
* `baseUnmarshaller.withDefaultValue`
* `baseUnmarshaller.mapWithCharset` (only available for FromEntityUnmarshallers)
* `baseUnmarshaller.forContentTypes` (only available for FromEntityUnmarshallers)

这些方法的签名很好的解释了其作用。

## 使用 Unmarshaller

在 Akka HTTP 很多场景下，Unmarshaller 是隐式使用的，比如当使用 [路由 DSL](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/index.html) 访问请求的 [实体](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/marshalling-directives/entity.html) 时。

当然也可以显式使用 Unmarshalling 设施，在测试中这很有用。直接使用时可以借助 `akka.http.scaladsl.marshalling.Unmarshal` 对象：

```scala
import akka.http.scaladsl.unmarshalling.Unmarshal
import system.dispatcher // Optional ExecutionContext (default from Materializer)
implicit val materializer: Materializer = ActorMaterializer()

import scala.concurrent.Await
import scala.concurrent.duration._

val intFuture = Unmarshal("42").to[Int]
val int = Await.result(intFuture, 1.second) // don't block in non-test code!
int shouldEqual 42

val boolFuture = Unmarshal("off").to[Boolean]
val bool = Await.result(boolFuture, 1.second) // don't block in non-test code!
bool shouldBe false
```
