# 消费 HTTP 服务（客户端）

Akka HTTP 所有客户端功能，即消费其他端点提供的基于 HTTP 的服务的功能，都由 `akka-http-core` 模块提供。

建议先阅读 [请求、响应实体的流式本质](../streaming_nature/introduction.md) 一节，该节讲解了底层的全栈流的概念，这对来自非 **streaming first** HTTP 客户端背景的用户可能有点出乎意料。

根据应用需要，可以选择 3 种不同抽象层次的 API：

## [请求层次的客户端 API](./Request-Level_Client-Side_API.md)

Akka HTTP 负责所有连接管理，大多数用户场景推荐使用。

## [主机层次的客户端 API](./Host-Level_Client-Side_API.md)

对与每个特定主机/端口端点，Akka HTTP 都会维护一个连接池。Recommended when the user can supply a `Source[HttpRequest, NotUsed]` with requests to run against a single host over multiple pooled connections.

## [连接层次的客户端 API](./Connection-Level_Client-Side_API.md)

用于完全控制何时打开、何时关闭连接，以及请求调度，仅推荐用于特定场景。

无论选择哪种抽象层次的 API，都可以同时与不同层次的 API 进行交互。Akka HTTP 可以轻松处理针对一个主机、或者多个主机的数千个并发连接。
