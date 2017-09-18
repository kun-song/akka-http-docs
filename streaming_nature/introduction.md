# 请求、响应实体的流式本质

Akka HTTP 是彻头彻尾的流式风格，这意味着 Akka Streams 提供的 {% em %}背压机制{% endem %} 将作用于每一层：从 TCP 层，到 HTTP 层，直到用户可见的 `HttpRequest` 和 `HttpResponse`，及其 `HttpEntity`。

如果你习惯于非流式 / 非响应式的 HTTP 客户端，则可能会感到惊讶。具体来说，{% em %}lack of consumption of the HTTP Entity, is signaled as back-pressure to the other side of the connection.{% endem %}，该特性允许应用只管消费实体，Akka HTTP 将背压服务端或客户端，避免应用在内存中产生不必要的实体缓存，从而压垮应用。

>{% em color="red" %}警告{% endem %}

>Akka HTTP 强制消费（或丢弃）请求实体！若实体既没有被消费，也没有被丢弃，则 Akka HTTP 认为该实体应该保持背压，进而通过背压机制，停止数据流入。无论 `HttpResponse` 的状态码是多少，客户端都应该消费响应实体。
