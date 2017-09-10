# JSON 支持

Akka HTTP 的序列化、反序列化机制使领域对象与 JSON 之间可以无缝转换。`akka-http-spray-json` 模块提供了对 [spray-json](https://github.com/spray/spray-json) 开箱即用的集成，与其他 JSON 库的集成由社区提供，查看 [当前社区对 Akka HTTP 的扩展列表](http://akka.io/community/?_ga=2.180381987.540013561.1505315916-1054048226.1500780092#extensions-to-akka-http)。

## spray-json 支持

对于类型 `T`，若存在隐式的 `spray.json.RootJsonReader` 和/或 `spray.json.RootJsonWriter`，则 [SprayJsonSupport](http://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/marshallers/sprayjson/SprayJsonSupport.html) 特质即可提供 `FromEntityUnmarshaller[T]` 和 `ToEntityMarshaller[T]`。

要使用 `spray-json` 自动进行 JSON 转换，需要增加如下依赖：

```scala
"com.typesafe.akka" %% "akka-http-spray-json" % "10.0.10
```

其次，为目标类型提供一个当前作用域可见的 `RootJsonFormat[T]`，更多详细内容，参考 [spray-json](https://github.com/spray/spray-json) 文档。

最后，像下面例子那样从 `SprayJsonSupport` 中直接导入隐式函数 `FromEntityUnmarshaller[T]` 和 `ToEntityMarshaller[T]`，或者将 `akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport` 特质混入自定义的 JSON 支持模块中。

做完这些后，JSON 与类型 `T` 之间的序列化、反序列化即可良好、透明地工作。

```scala
import akka.http.scaladsl.server.Directives
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport
import spray.json._

// domain model
final case class Item(name: String, id: Long)
final case class Order(items: List[Item])

// collect your json format instances into a support trait:
trait JsonSupport extends SprayJsonSupport with DefaultJsonProtocol {
  implicit val itemFormat = jsonFormat2(Item)
  implicit val orderFormat = jsonFormat1(Order) // contains List[Item]
}

// use it wherever json (un)marshalling is needed
class MyJsonService extends Directives with JsonSupport {

  // format: OFF
  val route =
    get {
      pathSingleSlash {
        complete(Item("thing", 42)) // will render as JSON
      }
    } ~
    post {
      entity(as[Order]) { order => // will unmarshal JSON to Order
        val itemsCount = order.items.size
        val itemNames = order.items.map(_.name).mkString(", ")
        complete(s"Ordered $itemsCount items: $itemNames")
      }
    }
  // format: ON
}
```

### 优美输出

默认情况下，spray-json 借助 `CompactPrinter` 进行隐式转化，将类型序列化为紧凑输出的 JSON：

```scala
implicit def sprayJsonMarshallerConverter[T](writer: RootJsonWriter[T])(implicit printer: JsonPrinter = CompactPrinter): ToEntityMarshaller[T] =
  sprayJsonMarshaller[T](writer, printer)
```

若要转化为美观输出的 JSON，需要把 `PrettyPrinter` 引入当前作用域，以便执行隐式转换。

```scala
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport._
import spray.json._

// domain model
final case class PrettyPrintedItem(name: String, id: Long)

object PrettyJsonFormatSupport {
  import DefaultJsonProtocol._
  implicit val printer = PrettyPrinter
  implicit val prettyPrintedItemFormat = jsonFormat2(PrettyPrintedItem)
}

// use it wherever json (un)marshalling is needed
class MyJsonService extends Directives {
  import PrettyJsonFormatSupport._

  // format: OFF
  val route =
    get {
      pathSingleSlash {
        complete {
          PrettyPrintedItem("akka", 42) // will render as JSON
        }
      }
    }
  // format: ON
}

val service = new MyJsonService

// verify the pretty printed JSON
Get("/") ~> service.route ~> check {
  responseAs[String] shouldEqual
    """{""" + "\n" +
    """  "name": "akka",""" + "\n" +
    """  "id": 42""" + "\n" +
    """}"""
}
```

关于更多 spray-json 的使用细节，请参考其 [文档](https://github.com/spray/spray-json)。
