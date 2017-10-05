# 高层服务端 API

除 [底层服务端 API](../low_server/Introduction.md) 外，Akka HTTP 提供非常灵活的 *路由 DSL*，用于优雅定义 RESTful web service，补充了底层 API 的空档，提供典型 Web 服务器、框架的大部分功能，比如 URI 解构、内容协商、静态内容服务。

>{% em color="red" %}注意{% endem %}

>建议先阅读 [请求、响应实体的流式本质](../streaming_nature/introduction.md) 一节，该节讲解了底层的全栈流的概念，这对来自非 **streaming first** HTTP 客户端背景的用户可能有点出乎意料。
