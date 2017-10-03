# 单独使用 HTTP 层

因为 Akka HTTP 基于 Reactive Streams，所以其余底层 TCP 接口完全解耦。大部分场景下，该特性并无大用，但有些场景下，若能够在非网络数据（非 TCP 连接到来的数据）上使用 HTTP 层，会很有用处，比如测试、调试等。

在服务端，单独的 HTTP 层构成的 `BidiFlow` 如下所示：

```scala
/**
 * The type of the server-side HTTP layer as a stand-alone BidiFlow
 * that can be put atop the TCP layer to form an HTTP server.
 *
 * {{{
 *                +------+
 * HttpResponse ~>|      |~> SslTlsOutbound
 *                | bidi |
 * HttpRequest  <~|      |<~ SslTlsInbound
 *                +------+
 * }}}
 */
type ServerLayer = BidiFlow[HttpResponse, SslTlsOutbound, SslTlsInbound, HttpRequest, NotUsed]
```

可使用 `Http().serverLayer` 的两个重载方法创建 `Http.ServerLayer` 实例，该方法也允许一定程度的配置。
