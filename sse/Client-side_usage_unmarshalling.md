# 客户端用法：反序列化

为能将事件流反序列化为 `Source[ServerSentEvent, NotUsed]`，需要引入 `EventStreamUnmarshalling` 定义的 `FromEntityUnmarshaller[Source[ServerSentEvent, NotUsed]]` 到当前作用域：

```scala
import akka.http.scaladsl.unmarshalling.sse.EventStreamUnmarshalling._

Http()
  .singleRequest(Get("http://localhost:8000/events"))
  .flatMap(Unmarshal(_).to[Source[ServerSentEvent, NotUsed]])
  .foreach(_.runForeach(println))
```

若要可靠的持久订阅事件流，则可以使用 Alpakka 提供 [EventSource](http://developer.lightbend.com/docs/alpakka/current/sse.html?_ga=2.181856999.1984122494.1507379879-2038150703.1489590101) 连接器，其会用上次连接的最后一个事件的 ID 自动重连。
