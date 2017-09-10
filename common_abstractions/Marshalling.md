# 序列化

Marshalling 是将高层抽象（对象）结构，转化为某种低层抽象形式（wire format）的过程，也被称为 serialization 或 picking。

>译者注：
1. 本译本将 Marshalling 翻译为序列化。
2. 关于 'wire format' 可以参考 RednaxelaFX 的这篇文章：[以Python为例讨论高级编程语言程序的wire format与校验](http://rednaxelafx.iteye.com/blog/382429)。

在 Akka HTTP 中，序列化是将类型 `T` 的对象，转化为某低层目标类型，例如 `MessageEntity`（该类为 HTTP 请求或响应的 'entity body' 部分）或完整的 `HttpRequest`、`HttpResponse`。

## 基本设计

使用 `Marshaller[A, B]` 将类型 `A` 的实例序列化为类型 `B` 的实例。Akka HTTP 预定义了一些常用 Marshaller 的别名：

```scala
type ToEntityMarshaller[T] = Marshaller[T, MessageEntity]
type ToByteStringMarshaller[T] = Marshaller[T, ByteString]
type ToHeadersAndEntityMarshaller[T] = Marshaller[T, (immutable.Seq[HttpHeader], MessageEntity)]
type ToResponseMarshaller[T] = Marshaller[T, HttpResponse]
type ToRequestMarshaller[T] = Marshaller[T, HttpRequest]
```

可能与你的猜想不同，`Marshaller[A, B]` 并非简单的 `A => B` 函数，而是 `A => Future[List[Marshalling[B]]]` 函数。让我们来剖析下该函数，以便理解 Marshaller 为何要这样设计。给定类型 `A` 的实例，计算 `Marshaller[A, B]` 将得到 a `Future` of `List` of `Marshalling[B]`：

1. A `Future`
  * Marshaller 不要求以 {% em %}同步{% endem %} 方式计算结果，因此返回一个 `Future`，以便以 {% em %}异步{% endem %} 方式进行序列化。
2. of `List`
  * Marshaller 可以返回多个 `A` 实例的目标表示形式，内容协商将决定实际发送哪个。例如 `ToEntityMarshaller[OrderConfirmation]` 可能提供 JSON 和 XML 两种表示形式，客户端通过 `Accept` 请求头表达自己的喜好，若客户端无 `Accept` 头部值，则选择顺序第一的表示形式。
3. of `Marshalling[B]`
  * Marshaller 没有直接返回 `B` 实例，而是返回 `Marshalling[B]`，在真正的序列化被触发前，通过它可以查询 Marshaller 最终计算结果的 `MediaType` 和 `HttpCharset`。该设计可用于内容协商，并且将目标对象的实际构建过程，延迟到真正需要那一刻。

下面是 `Marshalling` 的定义：

```scala
/**
 * Describes one possible option for marshalling a given value.
 */
sealed trait Marshalling[+A] {
  def map[B](f: A ⇒ B): Marshalling[B]

  /**
   * Converts this marshalling to an opaque marshalling, i.e. a marshalling result that
   * does not take part in content type negotiation. The given charset is used if this
   * instance is a `WithOpenCharset` marshalling.
   */
  def toOpaque(charset: HttpCharset): Marshalling[A]
}

object Marshalling {

  /**
   * A Marshalling to a specific [[akka.http.scaladsl.model.ContentType]].
   */
  final case class WithFixedContentType[A](
    contentType: ContentType,
    marshal:     () ⇒ A) extends Marshalling[A] {
    def map[B](f: A ⇒ B): WithFixedContentType[B] = copy(marshal = () ⇒ f(marshal()))
    def toOpaque(charset: HttpCharset): Marshalling[A] = Opaque(marshal)
  }

  /**
   * A Marshalling to a specific [[akka.http.scaladsl.model.MediaType]] with a flexible charset.
   */
  final case class WithOpenCharset[A](
    mediaType: MediaType.WithOpenCharset,
    marshal:   HttpCharset ⇒ A) extends Marshalling[A] {
    def map[B](f: A ⇒ B): WithOpenCharset[B] = copy(marshal = cs ⇒ f(marshal(cs)))
    def toOpaque(charset: HttpCharset): Marshalling[A] = Opaque(() ⇒ marshal(charset))
  }

  /**
   * A Marshalling to an unknown MediaType and charset.
   * Circumvents content negotiation.
   */
  final case class Opaque[A](marshal: () ⇒ A) extends Marshalling[A] {
    def map[B](f: A ⇒ B): Opaque[B] = copy(marshal = () ⇒ f(marshal()))
    def toOpaque(charset: HttpCharset): Marshalling[A] = this
  }
}
```

## 预定义的 Marshaller

Akka HTTP 为大多数常见类型，预定义了一批 Marshallers：

* [PredefinedToEntityMarshallers](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/marshalling/PredefinedToEntityMarshallers.html)
  + `Array[Byte]`
  + `ByteString`
  + `Array[Char]`
  + `String`
  + `akka.http.scaladsl.model.FormData`
  + `akka.http.scaladsl.model.MessageEntity`
  + `T <: akka.http.scaladsl.model.Multipart`
* [PredefinedToResponseMarshallers](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/marshalling/PredefinedToResponseMarshallers.html)
  + `T`, 若 `ToEntityMarshaller[T]` 可用
  + `HttpResponse`
  + `StatusCode`
  + `(StatusCode, T)`，若 `ToEntityMarshaller[T]` 可用
  + `(Int, T)`，若 `ToEntityMarshaller[T]` 可用
  + `(StatusCode, immutable.Seq[HttpHeader], T)`，若 `ToEntityMarshaller[T]` 可用
  + `(Int, immutable.Seq[HttpHeader], T)`，若 `ToEntityMarshaller[T]` 可用
* [PredefinedToRequestMarshallers](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/marshalling/PredefinedToRequestMarshallers.html)
  + `HttpRequest`
  + `Uri`
  + `(HttpMethod, Uri, T)`，若 `ToEntityMarshaller[T]` 可用
  + `(HttpMethod, Uri, immutable.Seq[HttpHeader], T)`，若 `ToEntityMarshaller[T]` 可用
* [GenericMarshallers](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/marshalling/GenericMarshallers.html)
  + `Marshaller[Throwable, T]`
  + `Marshaller[Option[A], B]``, if a `Marshaller[A, B]`` and an `EmptyValue[B]`` is available
  + `Marshaller[Either[A1, A2], B]``, if a `Marshaller[A1, B]`` and a `Marshaller[A2, B]`` is available
  + `Marshaller[Future[A], B]`, if a `Marshaller[A, B]` is available
  + `Marshaller[Try[A], B]`, if a `Marshaller[A, B]` is available

## 隐式解析

Akka HTTP 的序列化设施基于 {% em %}type-class{% endem %} 方式，这意味着从类型 `A` 到类型 `B` 的 `Marshaller` 必须隐式可用（to be available implicitly）。

大多数 Akka HTTP 预定义的 Marshallers 由 `Marshaller` 特质的伴生对象提供，即它们随时可用，不需要显式导入。通过在本地作用域自定义 Marshaller 可以覆盖默认版本。

## 自定义 Marshaller

Akka HTTP 提供了一些便利工具，用于构建用于特定类型的 Marshaller。在开始构建之前，首先确定要构建的 Marshaller 的种类。若仅需要生产出 `MessageEntity`，则应该使用 `ToEntityMarshaller[T]`，它既可用于客户端，也可用于服务端，因为若存在 `MessageEntity` 存在，则可自动创建 `ToResponseMarshaller[T]` 和 `ToRequestMarshaller[T]`。

但是，若需要设置状态码、请求方法、请求 URI 或任意头部，则 `ToEntityMarshaller[T]` 无法满足，需要使用 `ToRequestMarshaller[T]` 或者 `ToResponseMarshaller[T]`。

自定义 Marshaller 不需要直接手动实现 `Marshaller` 特质，可以使用 `Marshaller` 伴生对象中的构建辅助函数：

```scala
object Marshaller
  extends GenericMarshallers
  with PredefinedToEntityMarshallers
  with PredefinedToResponseMarshallers
  with PredefinedToRequestMarshallers {

  /**
   * Creates a [[Marshaller]] from the given function.
   */
  def apply[A, B](f: ExecutionContext ⇒ A ⇒ Future[List[Marshalling[B]]]): Marshaller[A, B] =
    new Marshaller[A, B] {
      def apply(value: A)(implicit ec: ExecutionContext) =
        try f(ec)(value)
        catch { case NonFatal(e) ⇒ FastFuture.failed(e) }
    }

  /**
   * Helper for creating a [[Marshaller]] using the given function.
   */
  def strict[A, B](f: A ⇒ Marshalling[B]): Marshaller[A, B] =
    Marshaller { _ ⇒ a ⇒ FastFuture.successful(f(a) :: Nil) }

  /**
   * Helper for creating a "super-marshaller" from a number of "sub-marshallers".
   * Content-negotiation determines, which "sub-marshaller" eventually gets to do the job.
   *
   * Please note that all marshallers will actualy be invoked in order to get the Marshalling object
   * out of them, and later decide which of the marshallings should be returned. This is by-design,
   * however in ticket as discussed in ticket https://github.com/akka/akka-http/issues/243 it MAY be
   * changed in later versions of Akka HTTP.
   */
  def oneOf[A, B](marshallers: Marshaller[A, B]*): Marshaller[A, B] =
    Marshaller { implicit ec ⇒ a ⇒ FastFuture.sequence(marshallers.map(_(a))).fast.map(_.flatten.toList) }

  /**
   * Helper for creating a "super-marshaller" from a number of values and a function producing "sub-marshallers"
   * from these values. Content-negotiation determines, which "sub-marshaller" eventually gets to do the job.
   *
   * Please note that all marshallers will actualy be invoked in order to get the Marshalling object
   * out of them, and later decide which of the marshallings should be returned. This is by-design,
   * however in ticket as discussed in ticket https://github.com/akka/akka-http/issues/243 it MAY be
   * changed in later versions of Akka HTTP.
   */
  def oneOf[T, A, B](values: T*)(f: T ⇒ Marshaller[A, B]): Marshaller[A, B] =
    oneOf(values map f: _*)

  /**
   * Helper for creating a synchronous [[Marshaller]] to content with a fixed charset from the given function.
   */
  def withFixedContentType[A, B](contentType: ContentType)(marshal: A ⇒ B): Marshaller[A, B] =
    new Marshaller[A, B] {
      def apply(value: A)(implicit ec: ExecutionContext) =
        try FastFuture.successful {
          Marshalling.WithFixedContentType(contentType, () ⇒ marshal(value)) :: Nil
        } catch {
          case NonFatal(e) ⇒ FastFuture.failed(e)
        }

      override def compose[C](f: C ⇒ A): Marshaller[C, B] =
        Marshaller.withFixedContentType(contentType)(marshal compose f)
    }

  /**
   * Helper for creating a synchronous [[Marshaller]] to content with a negotiable charset from the given function.
   */
  def withOpenCharset[A, B](mediaType: MediaType.WithOpenCharset)(marshal: (A, HttpCharset) ⇒ B): Marshaller[A, B] =
    new Marshaller[A, B] {
      def apply(value: A)(implicit ec: ExecutionContext) =
        try FastFuture.successful {
          Marshalling.WithOpenCharset(mediaType, charset ⇒ marshal(value, charset)) :: Nil
        } catch {
          case NonFatal(e) ⇒ FastFuture.failed(e)
        }

      override def compose[C](f: C ⇒ A): Marshaller[C, B] =
        Marshaller.withOpenCharset(mediaType)((c: C, hc: HttpCharset) ⇒ marshal(f(c), hc))
    }

  /**
   * Helper for creating a synchronous [[Marshaller]] to non-negotiable content from the given function.
   */
  def opaque[A, B](marshal: A ⇒ B): Marshaller[A, B] =
    strict { value ⇒ Marshalling.Opaque(() ⇒ marshal(value)) }

  /**
   * Helper for creating a [[Marshaller]] combined of the provided `marshal` function
   * and an implicit Marshaller which is able to produce the required final type.
   */
  def combined[A, B, C](marshal: A ⇒ B)(implicit m2: Marshaller[B, C]): Marshaller[A, C] =
    Marshaller[A, C] { ec ⇒ a ⇒ m2.compose(marshal).apply(a)(ec) }
}
```

## 继承 Marshaller

利用已存在的 Marshaller 来构建自己的 Marshaller 会节省时间，方法是封装已存在的 Marshaller，并将其结果 “重定向” 到自己的类型。

此处，封装 Marshaller 意味着：

* 在输入到达 Marshaller 之前，对输入进行转换
* 对输出进行转换

转换输出，可以使用 `baseMarshaller.map`，转换输入，可以使用：

* `baseMarshaller.compose`
* `baseMarshaller.composeWithEC`
* `baseMarshaller.wrap`
* `baseMarshaller.wrapWithEC`

其中 `compose` 与用于组合函数时作用相同；`wrap` 类似组合函数，但可以修改序列化目标的 `ContentType`；`...WithEC` 在内部可以访问 `ExecutionContext`，从而避免对当前作用域 `ExecutionContext` 的依赖。

## 使用 Marshaller

在 Akka HTTP 很多场景下，Marshaller 是隐式使用的，比如当使用 [路由 DSL](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/index.html) 定义如何 [complete](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/route-directives/complete.html) 请求时。

当然也可以显式使用 Marshalling 设施，在测试中这很有用。直接使用时可以借助 `akka.http.scaladsl.marshalling.Marshal` 对象：

```scala
import scala.concurrent.Await
import scala.concurrent.duration._
import akka.http.scaladsl.marshalling.Marshal
import akka.http.scaladsl.model._

import system.dispatcher // ExecutionContext

val string = "Yeah"
val entityFuture = Marshal(string).to[MessageEntity]
val entity = Await.result(entityFuture, 1.second) // don't block in non-test code!
entity.contentType shouldEqual ContentTypes.`text/plain(UTF-8)`

val errorMsg = "Easy, pal!"
val responseFuture = Marshal(420 -> errorMsg).to[HttpResponse]
val response = Await.result(responseFuture, 1.second) // don't block in non-test code!
response.status shouldEqual StatusCodes.EnhanceYourCalm
response.entity.contentType shouldEqual ContentTypes.`text/plain(UTF-8)`

val request = HttpRequest(headers = List(headers.Accept(MediaTypes.`application/json`)))
val responseText = "Plaintext"
val respFuture = Marshal(responseText).toResponseFor(request) // with content negotiation!
a[Marshal.UnacceptableResponseContentTypeException] should be thrownBy {
  Await.result(respFuture, 1.second) // client requested JSON, we only have text/plain!
}
```
