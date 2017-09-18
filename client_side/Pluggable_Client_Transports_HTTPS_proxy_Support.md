# 可插拔的客户端 Transports/HTTP(S) 代理支持

客户端侧支持可插拔的底层传输机制（不稳定）。客户端传输通过 [akka.http.scaladsl.ClientTransport](https://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/ClientTransport.html) 实例表示。

```scala
@ApiMayChange
trait ClientTransport {
  def connectTo(host: String, port: Int, settings: ClientConnectionSettings)(implicit system: ActorSystem): Flow[ByteString, ByteString, Future[OutgoingConnection]]
}
```

传输实现（`ClientTransport`）定义了客户端基础设施如何与给定主机通信。

## 配置 `ClientTransport`

HTTP 客户端抽象层次不同，`ClientTransport` 配置也不同。目前，只支持代码配置（不支持配置文件）。没有预定义为每个主机选择不同 Transports 的方式（但通过实现 `ClientTransport`，很容易定义任何策略）。

### 连接池

`ConnectionPoolSettings` 类可以为任意连接池方法设置自定义的 Transport。使用 `ConnectionPoolSettings.withTransport` 方法配置 Transport，并将其传递到连接池方法，例如 `Http().singleRequest`，`Http().superPool` 或 `Http().cachedHostConnectionPool`。

### 单个连接

将自定义的 Transport 传递给 `Http().outgoingConnectionUsingTransport` 方法，可以为单个 HTTP 连接设置 Transport。

## 预定义 Transports

### TCP

默认 Transport 为 `ClientTransport.TCP`，它为目标主机打开一个 TCP 连接。

### HTTP(S) 代理

该 transport 通过 HTTP(S) 代理连接到目标服务器。HTTP(S) 代理使用 HTTP `CONNECT`（[RFC 7231 Section 4.3.6](https://tools.ietf.org/html/rfc7231#section-4.3.6)）方法创建通向目标服务器的通道。代理本身应该透明地将数据转发给目标服务器，因此端到端的加密依然有效（若 TLS 失效，则代理可能会不接受数据 if TLS breaks, then the proxy might be fussing with your data）。

This approach is commonly used to securely proxy requests to HTTPS endpoints. In theory it could also be used to proxy requests targeting HTTP endpoints, but we have not yet found a proxy that in fact allows this.

使用 `ClientTransport.httpsProxy(proxyAddress)` 方法实例化 HTTP(S) 代理 transport。

### `Http().singleRequest` 中使用 HTTP(S) 代理

要在 `singleRequest` 方法上使用 HTTP 代理，只需要添加代理配置，并将其传递给 `singleRequest` 方法即可。

```scala
import java.net.InetSocketAddress

import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import akka.http.scaladsl.{ ClientTransport, Http }

implicit val system = ActorSystem()
implicit val materializer = ActorMaterializer()

val proxyHost = "localhost"
val proxyPort = 8888

val httpsProxyTransport = ClientTransport.httpsProxy(InetSocketAddress.createUnresolved(proxyHost, proxyPort))

val settings = ConnectionPoolSettings(system).withTransport(httpsProxyTransport)
Http().singleRequest(HttpRequest(uri = "https://google.com"), settings = settings)
```

### 需要鉴权的 HTTP(S) 代理

若 HTTP(S) 代理需要鉴权，则向代理发出 `CONNECT` 请求时，需要提供 `HttpCredentials`：

```scala
import akka.http.scaladsl.model.headers

val proxyAddress = InetSocketAddress.createUnresolved(proxyHost, proxyPort)
val auth = headers.BasicHttpCredentials("proxy-user", "secret-proxy-pass-dont-tell-anyone")

val httpsProxyTransport = ClientTransport.httpsProxy(proxyAddress, auth)

val settings = ConnectionPoolSettings(system).withTransport(httpsProxyTransport)
Http().singleRequest(HttpRequest(uri = "http://akka.io"), settings = settings)
```

## 自定义 Transports

通过实现 `ClientTransport.connectTo` 方法来自定义 `ClientTransport`。

下面提供两个自定义的思路（或将来会默认提供）：

* SSH 通道 transport：通过 SSH 通道连接目标主机
* Per-host configurable transport：允许为每个目标主机配置不同 transport
