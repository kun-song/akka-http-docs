# HTTP 服务器路由 DSL

`Akka HTTP` 的高层路由 API 提供了描述 HTTP **路由** 的 DSL，以及路由处理规则。路由由一个或多个 `Directive` 嵌套组成，每个 `Directive` 都会缩小请求范围，最后得到一个特定类型的请求。

例如，某路由起始于匹配请求路径，只有路径为 `/hello` 时才会匹配，然后缩小该请求范围，只处理 HTTP `get` 请求，最后以字符串字面值 `complete`，则该请求的响应为 HTTP OK，响应体为前面的字符串。

请求体、响应体在网络传输格式与应用对象之间的转换在 `marshallers` 中完成，与路由定义完全解耦，`marshallers` 使用 `magnet` 模式隐式引入，这意味着，可以使用任何对象 `complete` 请求，只要该对象在作用域中有合适的隐式 `marshaller`。

`Akka HTTP` 提供了简单对象（比如 `String` 或者 `ByteString`）的默认 `marshaller`，也可以自定义 `marshaller`（比如 JSON `marshaller`）。另有模块提供了基于 `spray-json` 的 JSON 序列化（更多细节，查看[JSON 支持](http://doc.akka.io/docs/akka-http/current/scala/http/common/json-support.html)）。

使用路由 DSL 创建 `Route`，然后将其 *绑定* 到一个端口，就可以开始接收 HTTP 请求了：

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

一个常见场景为使用领域模型对象响应请求，此时存在将该对象转化为 JSON 的 `marshaller`。下面的例子由两个路由组成，第一个查询异步数据库，并将查询结果 `Future[Option[Item]]` 序列化（marshall）为 JSON 响应；第二个从进入的请求中反序列化（unmarshall）获取 `Order`，并将其存入数据库，成功后以 OK 作为响应：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.stream.ActorMaterializer
import akka.Done
import akka.http.scaladsl.server.Route
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.model.StatusCodes
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport._
import spray.json.DefaultJsonProtocol._

import scala.io.StdIn

import scala.concurrent.Future

object WebServer {

  // domain model
  final case class Item(name: String, id: Long)
  final case class Order(items: List[Item])

  // formats for unmarshalling and marshalling
  implicit val itemFormat = jsonFormat2(Item)
  implicit val orderFormat = jsonFormat1(Order)

  // (fake) async database query api
  def fetchItem(itemId: Long): Future[Option[Item]] = ???
  def saveOrder(order: Order): Future[Done] = ???

  def main(args: Array[String]) {

    // needed to run the route
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    // needed for the future map/flatmap in the end
    implicit val executionContext = system.dispatcher

    val route: Route =
      get {
        pathPrefix("item" / LongNumber) { id =>
          // there might be no item for a given id
          val maybeItem: Future[Option[Item]] = fetchItem(id)

          onSuccess(maybeItem) {
            case Some(item) => complete(item)
            case None       => complete(StatusCodes.NotFound)
          }
        }
      } ~
        post {
          path("create-order") {
            entity(as[Order]) { order =>
              val saved: Future[Done] = saveOrder(order)
              onComplete(saved) { done =>
                complete("order created")
              }
            }
          }
        }

    val bindingFuture = Http().bindAndHandle(route, "localhost", 8080)
    println(s"Server online at http://localhost:8080/\nPress RETURN to stop...")
    StdIn.readLine() // let it run until user presses return
    bindingFuture
      .flatMap(_.unbind()) // trigger unbinding from the port
      .onComplete(_ ⇒ system.terminate()) // and shutdown when done

  }
}
```

上例中的序列化、反序列化逻辑由 `spray-json` 提供（更多细节，请查看[JSON 支持](http://doc.akka.io/docs/akka-http/current/scala/http/common/json-support.html)）。

Akka HTTP 另一强大之处在于其以流式数据为核心，即请求体、响应体都都可以通过服务端进行流式处理，即使面对超大的请求或响应，内存使用依然可以保持稳定。客户端会背压 **流式响应** 使服务端数据推送速度不超过客户端处理能力；服务端会背压 **流式响应** 使服务端可以决定客户端发送请求数据的速度（译者注：保证服务端不会被撑爆）。

下面例子在客户端能够接受的范围内，服务端发送随机数流：

```scala
import akka.actor.ActorSystem
import akka.stream.scaladsl._
import akka.util.ByteString
import akka.http.scaladsl.Http
import akka.http.scaladsl.model.{HttpEntity, ContentTypes}
import akka.http.scaladsl.server.Directives._
import akka.stream.ActorMaterializer
import scala.util.Random
import scala.io.StdIn

object WebServer {

  def main(args: Array[String]) {

    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    // needed for the future flatMap/onComplete in the end
    implicit val executionContext = system.dispatcher

    // streams are re-usable so we can define it here
    // and use it for every request
    val numbers = Source.fromIterator(() =>
      Iterator.continually(Random.nextInt()))

    val route =
      path("random") {
        get {
          complete(
            HttpEntity(
              ContentTypes.`text/plain(UTF-8)`,
              // transform each number to a chunk of bytes
              numbers.map(n => ByteString(s"$n\n"))
            )
          )
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

使用网速较慢的 HTTP 客户端连接该服务时，会背压服务端，使随机数按需生成，保证服务端保持稳定的内存使用。使用 `curl --limit-rate 50b 127.0.0.1:8080/random` 观察该效果。

Akka HTTP 路由与 actors 交互非常便利，下面例子中一个路由以 `fire-and-forget` 模式发出投标，另一个路由则包含一次与 actor 的请求、响应交互。actor 返回的响应被序列化为 JSON 返回到客户端。

```scala
import akka.actor.{Actor, ActorSystem, Props, ActorLogging}
import akka.http.scaladsl.Http
import akka.http.scaladsl.model.StatusCodes
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.marshallers.sprayjson.SprayJsonSupport._
import akka.pattern.ask
import akka.stream.ActorMaterializer
import akka.util.Timeout
import spray.json.DefaultJsonProtocol._
import scala.concurrent.duration._
import scala.io.StdIn

object WebServer {

  case class Bid(userId: String, offer: Int)
  case object GetBids
  case class Bids(bids: List[Bid])

  class Auction extends Actor with ActorLogging {
    var bids = List.empty[Bid]
    def receive = {
      case bid @ Bid(userId, offer) =>
        bids = bids :+ bid
        log.info(s"Bid complete: $userId, $offer")
      case GetBids => sender() ! Bids(bids)
      case _ => log.info("Invalid message")
    }
  }

  // these are from spray-json
  implicit val bidFormat = jsonFormat2(Bid)
  implicit val bidsFormat = jsonFormat1(Bids)

  def main(args: Array[String]) {
    implicit val system = ActorSystem()
    implicit val materializer = ActorMaterializer()
    // needed for the future flatMap/onComplete in the end
    implicit val executionContext = system.dispatcher

    val auction = system.actorOf(Props[Auction], "auction")

    val route =
      path("auction") {
        put {
          parameter("bid".as[Int], "user") { (bid, user) =>
            // place a bid, fire-and-forget
            auction ! Bid(user, bid)
            complete((StatusCodes.Accepted, "bid placed"))
          }
        } ~
        get {
          implicit val timeout: Timeout = 5.seconds

          // query the actor for the current auction state
          val bids: Future[Bids] = (auction ? GetBids).mapTo[Bids]
          complete(bids)
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

查阅 [高层服务端 API](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/index.html) 获取更多高层抽象 API 的细节。
