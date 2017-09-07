# Akka HTTP 组成模块

Akka HTTP 由以下 5 个模块组成：

#####**akka-http**

该模块包含高层抽象的功能，比如序列化与反序列化、压缩与解压、路由 DSL 等，推荐使用该模块构建 HTTP 服务器。更多细节，请查看 [High-level Server-Side API](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/index.html)。

#####**akka-http-core**

包含完整的底层（HTTP 及 WebSockets）服务端、客户端实现，更多细节，请分别查看 [Low-Level Server-Side API](http://doc.akka.io/docs/akka-http/current/scala/http/server-side/low-level-api.html) [Consuming HTTP-based Services (Client-Side)](http://doc.akka.io/docs/akka-http/current/scala/http/client-side/index.html)。

#####**akka-http-testkit**

包含测试工具、服务端实现的验证工具。

#####**akka-http-spray-json**

使用 `spary-json` 进行序列化、反序列化的胶水代码，更多内容查看 [JSON Support](http://doc.akka.io/docs/akka-http/current/scala/http/common/json-support.html)。

#####**akka-http-xml**

使用 `scala-xml` 进行序列化、反序列化的胶水代码，更多内容，请查看 [XML Support](http://doc.akka.io/docs/akka-http/current/scala/http/common/xml-support.html)。
