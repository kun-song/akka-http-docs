# 服务端用法：序列化

为以事件流响应 HTTP 请求，需要引入 `EventStreamMarshalling` 定义的 `ToResponseMarshaller[Source[ServeSentEvent, Any]]` 到当前作用域：

```scala
import akka.NotUsed
import akka.stream.scaladsl.Source

import akka.http.scaladsl.Http
import akka.http.scaladsl.unmarshalling.Unmarshal
import akka.http.scaladsl.model.sse.ServerSentEvent
import scala.concurrent.duration._

import java.time.LocalTime
import java.time.format.DateTimeFormatter.ISO_LOCAL_TIME

def route: Route = {
  import akka.http.scaladsl.marshalling.sse.EventStreamMarshalling._

  path("events") {
    get {
      complete {
        Source
          .tick(2.seconds, 2.seconds, NotUsed)
          .map(_ => LocalTime.now())
          .map(time => ServerSentEvent(ISO_LOCAL_TIME.format(time)))
          .keepAlive(1.second, () => ServerSentEvent.heartbeat)
      }
    }
  }
}
```
