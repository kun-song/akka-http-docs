# Akka HTTP 超时

Akka HTTP 提供了很多内置的超时机制，以保护服务免受恶意攻击、编程错误的危害。其中一部分是简单的配置选项（可通过代码覆盖），另外一部分是流式 API，可直接在用户代码中作为模式实现。

## 通用超时

### 空闲超时

`idle-timeout` 是一个全局配置，用来设置给定连接的最大不活跃时间，即若连接处于打开状态，但在超过 `idle-timeout` 时间段内，并无请求、响应写入，则自动关闭该连接。

该设置作用域任意连接，无论是服务端还是客户端，且可以各自独立配置：

```scala
akka.http.server.idle-timeout
akka.http.client.idle-timeout
akka.http.host-connection-pool.idle-timeout
akka.http.host-connection-pool.client.idle-timeout
```

>注意：对于客户端连接池，只有当连接池无挂起请求等待时，才开始计算空闲时间。

## 服务端超时

### 请求超时

请求超时限制了针对给定路由，服务端产生 `HttpResponse` 响应的最大时间，若该时间内无法产生响应，则服务端自动注“服务不可用”响应，并关闭该连接，以避免连接泄漏、连接无限等待（例如，若因为编程错误，Future 永远无法结束，永远无法产生响应）。

请求超时发生时，写入的默认 `HttpResponse` 类似下面：

```scala
HttpResponse(StatusCodes.ServiceUnavailable, entity = "The server was not able " +
  "to produce a timely response to your request.\r\nPlease try again in a short while!")
```

默认请求超时作用于全局范围内的所有路由，可以使用 `akka.http.server.request-timeout` 进行设置（默认为 20 秒）。

>注意：若同一客户端使用同一连接，发出多个请求（`R1, R2, R3, ...`），其中第 `n - th` 个触发了请求超时，则服务端将回复一个（服务不可用）响应，并关闭该连接，该连接上剩余的 `(n+1) - th` 个请求将无法处理。（译者注：{% em %}[HTTP pipelining](https://en.wikipedia.org/wiki/HTTP_pipelining){% endem %} 允许客户端发送多个请求，服务端将按照其接收请求的顺序发送响应，所以前 `th - 1` 个请求可以正常返回，第 `th` 个请求的响应为服务不可用，后面 `(n+1) - th` 个请求无法处理）。

指定路由的请求超时可在运行时，超过 [TimeoutDirectives](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/timeout-directives/index.html) 进行配置。

### 绑定超时

绑定超时是建立 TCP 连接（`Http().bind*`）的时间限制，可通过 `akka.http.server.bind-timeout` 配置。

### 逗留超时

逗留超时时间是，当所有数据发送到网络层后，HTTP 服务器继续保持连接开放的时间。该设置类似 SO_LINGER 套接字选项，但不仅包含操作系统层的套接字，而且包含 Akka IO / Akka Stream 的网络栈。该设置用于防止，当服务端认为连接已经结束时，客户端继续保持连接。

若网络层缓存（包括 Akka Stream / Akka IO 网络栈缓存）包含的数据，不足以在给定时间内发送到客户端，而此时服务端认为连接已经结束，则客户端可能遇到连接重置。

将 `linger-timeout` 设置为 `infinite` 可禁止自动关闭连接（有连接泄漏风险）。

## 客户端超时

### 连接超时

连接超时是建立 TCP 连接的时间限制，一般不需要修改，当连接在给定时间内无法建立时，通过配置连接超时，可以排除该连接。

通过 `akka.http.client.connecting-timeout` 对其进行设置。
