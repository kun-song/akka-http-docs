# 底层服务端 API

除 [HTTP Client](../client_side/Introduction.md) 外，Akka HTTP 在 [Akka Streams](https://doc.akka.io/docs/akka/2.5.6/scala/stream/index.html) 之上，提供一个嵌入式的、基于 [Reactive Streams](http://www.reactive-streams.org/) 的 HTTP/1.1 服务器。

目前支持如下特性：

* 完全支持 [HTTP persistent connections](https://en.wikipedia.org/wiki/HTTP_persistent_connection)
* 完全支持 [HTTP pipelining](https://en.wikipedia.org/wiki/HTTP_pipelining)
* 完全支持异步 HTTP 流，包括分块传输编码
* 可选的 SSL/TLS 加密
* WebSocket 支持

Akka HTTP 服务端的组件可以分为两层：

1. 基础的底层服务器实现，`akka-http-core` 模块
2. 高级功能，`akka-http` 模块

其中第 1 条专注于实现 HTTP/1.1 服务器的核心功能：

* 连接管理
* 消息、头部的解析、生成
* 超时管理（请求和连接）
* 响应排序（为提供透明的流水线支持）

典型 HTTP 服务器的其他非核心功能（比如请求路由、文件服务、压缩等）留给更高抽象层实现，它们不在 `akka-http-core` 中实现。这使服务器核心很小，且轻量级，便于理解和维护。

根据需求，可以使用底层 API，或者更高层的 [Routing DSL](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/index.html)，高层 DSL 将简化复杂服务逻辑的定义。

>注意

>建议先阅读 [请求、响应实体的流式本质](../streaming_nature/introduction.md) 一节，该节讲解了底层的全栈流的概念，这对来自非 **streaming first** HTTP 客户端背景的用户可能有点出乎意料。
