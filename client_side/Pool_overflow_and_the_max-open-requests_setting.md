# 连接池溢出与 max-open-requests 设置

[请求层 API](./Request-Level_Client-Side_API.md) 和 [主机层 API](./Host-Level_Client-Side_API.md) 底层都基于连接池，连接池针对每个主机，会打开有限数量的并发连接（`akka.http.host-connection-pool.max-connections` 设置），这会限制请求的速度。

当使用 [基于流的主机层 API](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/client-side/host-level.html#using-the-host-level-api-in-a-streaming-fashion) 时，其语义避免了（请求过多造成的）连接池溢出。另一方面，`Http().singleRequest()` 方法或使用相同的 `Http().cachedHostConnectionPool` 物质化了过多流时，都会产生新请求，若请求生产速度大于连接池的处理能力时，请求将开始排队。

该场景下，每个主机连接池，若连接数量不超过 `max-open-requests`，则请求将排队，以缓存短期的请求高峰。超出最大个数后，请求将立刻失败，抛出 `BuferOverflowException` 异常，其中携带类似如下的消息：

```
Exceeded configured max-open-requests value of ...
```

这经常发生于高负载场景下，或连接池的处理速度太慢，难以及时处理所有进入的请求时。

注意，即使连接池能够处理常规负载，短期停顿（服务端、网络、客户端）依然可以导致 **队列溢出**，因此需将其视为可预见条件，应用应具备处理该场景的能力。多数场景下，将连接池溢出与服务端 `503` 错误（常常用于服务端超载）视为相同，通常的处理方式为过段时间后，重试该请求。

## 连接池溢出的常见原因

连接池溢出的原因为 **请求到达速度** 大于 **请求处理速度**，很多原因会导致该问题（解决提示在括号中）：

* 服务端太慢（提升服务费性能）
* 网络太慢（提升网络性能）
* 客户端发出请求太快（若可能，减少请求创建速度）
* 客户端、服务端之间时延太高（使用更多并发连接，以隐藏时延）
* 请求速度有峰值（调节客户端以消除峰值，或增大 `max-open-requests` 以缓存短期峰值）
* 响应实体未被读取、未被丢弃（参考 [请求、响应实体的流式本质](../streaming_nature/introduction.md)）
* 部分请求速度太慢，阻塞连接池中的（其他请求的）连接（解释在下面）

最后一点需要解释一下，若部分请求大大慢于其他请求，例如，若请求是长时间运行的 *Server Sent Events*，则其将长时间阻塞连接池中的某条连接。若同一时间，有多个这样的连接，会导致（请求）饥饿，其他请求无法前进。为避免该问题，确保在专用连接（使用 [连接层 API](./Connection-Level_Client-Side_API.md)）上执行长时间请求。

## 为何只有 Akka HTTP 有该问题，而其他客户端没有？

很多 Java HTTP 客户端不会设置资源的使用限制，比如，部分客户端不使用请求队列，每当连接池中无空闲连接时，都会打开一个新连接。然而，这仅仅将问题从客户端转移到了服务端，而且过多连接会竞争带宽，从而降低网络性能。
