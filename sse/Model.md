# 模型

Akka HTTP 用 `Source[ServerSentEvent, NotUsed]` 表示事件流，其中 `ServerSentEvent` 是一个 *case class*，且包含如下只读属性：

* `data: String`：实际负载，可占用多行
* `eventType: Option[String]`：可选的修饰符，比如 *added* *removed* 等
* `id: Option[String]`：可选的标识符
* `retry: Option[Int]`：可选的重连延迟时间，单位 *milliseconds*

依照 SSE 标准，Akka HTTP 亦提供 `Last-Event-ID` 头部字段与 `text/event-stream` 媒体类型。
