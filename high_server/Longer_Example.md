# 复杂一点的例子

下例展示了 Akka HTTP 的路由定义，可以稍微感受下使用路由 DSL 定义真实 API 的样子：

```scala
import akka.actor.{ActorRef, ActorSystem}
import akka.http.scaladsl.coding.Deflate
import akka.http.scaladsl.marshalling.ToResponseMarshaller
import akka.http.scaladsl.model.StatusCodes.MovedPermanently
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.unmarshalling.FromRequestUnmarshaller
import akka.pattern.ask
import akka.stream.ActorMaterializer
import akka.util.Timeout

// types used by the API routes
type Money = Double // only for demo purposes, don't try this at home!
type TransactionResult = String
case class User(name: String)
case class Order(email: String, amount: Money)
case class Update(order: Order)
case class OrderItem(i: Int, os: Option[String], s: String)

// marshalling would usually be derived automatically using libraries
implicit val orderUM: FromRequestUnmarshaller[Order] = ???
implicit val orderM: ToResponseMarshaller[Order] = ???
implicit val orderSeqM: ToResponseMarshaller[Seq[Order]] = ???
implicit val timeout: Timeout = ??? // for actor asks
implicit val ec: ExecutionContext = ???
implicit val mat: ActorMaterializer = ???
implicit val sys: ActorSystem = ???

// backend entry points
def myAuthenticator: Authenticator[User] = ???
def retrieveOrdersFromDB: Seq[Order] = ???
def myDbActor: ActorRef = ???
def processOrderRequest(id: Int, complete: Order => Unit): Unit = ???

val route = {
  path("orders") {
    authenticateBasic(realm = "admin area", myAuthenticator) { user =>
      get {
        encodeResponseWith(Deflate) {
          complete {
            // marshal custom object with in-scope marshaller
            retrieveOrdersFromDB
          }
        }
      } ~
      post {
        // decompress gzipped or deflated requests if required
        decodeRequest {
          // unmarshal with in-scope unmarshaller
          entity(as[Order]) { order =>
            complete {
              // ... write order to DB
              "Order received"
            }
          }
        }
      }
    }
  } ~
  // extract URI path element as Int
  pathPrefix("order" / IntNumber) { orderId =>
    pathEnd {
      (put | parameter('method ! "put")) {
        // form extraction from multipart or www-url-encoded forms
        formFields(('email, 'total.as[Money])).as(Order) { order =>
          complete {
            // complete with serialized Future result
            (myDbActor ? Update(order)).mapTo[TransactionResult]
          }
        }
      } ~
      get {
        // debugging helper
        logRequest("GET-ORDER") {
          // use in-scope marshaller to create completer function
          completeWith(instanceOf[Order]) { completer =>
            // custom
            processOrderRequest(orderId, completer)
          }
        }
      }
    } ~
    path("items") {
      get {
        // parameters to case class extraction
        parameters(('size.as[Int], 'color ?, 'dangerous ? "no"))
          .as(OrderItem) { orderItem =>
            // ... route using case class instance created from
            // required and optional query parameters
            complete("") // hide
          }
      }
    }
  } ~
  pathPrefix("documentation") {
    // optionally compresses the response with Gzip or Deflate
    // if the client accepts compressed responses
    encodeResponse {
      // serve up static content from a JAR resource
      getFromResourceDirectory("docs")
    }
  } ~
  path("oldApi" / Remaining) { pathRest =>
    redirect("http://oldapi.example.com/" + pathRest, MovedPermanently)
  }
}
```
