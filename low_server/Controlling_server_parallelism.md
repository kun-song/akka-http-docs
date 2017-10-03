# 控制服务端并行

请求处理可以在两个维度并行，1）通过并行处理连接，2）通过 HTTP pipelining 在同一连接发送多个请求（不需要遵循发送请求、获取连接这样一问一答的形式）。两种场景中，都由客户端控制并行请求数量。为避免被过量请求撑垮，Akka HTTP 服务端可以限制并行处理的请求上限。

为限制同时开放的连接上限，可以使用 `akka.http.server.max-connections` 设置，该设置应用于所有 `Http.bindAndHandle*` 方法。若使用 `Http.bind` 方法，则连接被表示为 `Source[IncomingConnection, ...]`，可以使用 Akka Streams 的组合子，比如 `throttle` 或 `mapAsync`，对连接流施加背压。

通常不推荐使用 HTTP pipelining（[大多数浏览器禁止使用](https://en.wikipedia.org/w/index.php?title=HTTP_pipelining&oldid=700966692#Implementation_in_web_browsers)），但 Akka HTTP 还是全面支持 HTTP pipelining。使用 HTTP pipelining 时，请求限制体现在两个层面。第一，`akka.http.server.pipeline-limit` 限制给用户提供的 handler-flow 的请求数量不超过该配置。第二，handler-flow 自己可以施加任何 `throttle`。若使用 `Http.bindAndHandleSync` 或 `Http.bindAndHandleAsync` 方法，可以提供 `parallelism` 参数（默认为 1，即禁止 HTTP pipelining），以控制每个连接并发请求的数量。若使用 `Http.bindAndHandle` 或 `Http.bind`，则用户 handler-flow 通过背压决定自己可接受的并发请求数量。该场景下，可以使用 Akka Streams 的 `mapAsync` 组合子限制并发请求数。实际使用时，配置和手动 flow 设置两者中，限制更强者，将决定一个连接上的并发请求数。
