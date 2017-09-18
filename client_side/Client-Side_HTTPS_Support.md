# 客户端 HTTPS 支持

Akka HTTP 客户端、[服务端](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/server-side/server-https-support.html)均支持 TLS 加密。

配置加密的核心是 `HttpsConnectionContext`，可以通过静态方法 `ConnectionContext.https` 创建：

```scala
// ConnectionContext
def https(
  sslContext:          SSLContext,
  sslConfig:           Option[AkkaSSLConfig]         = None,
  enabledCipherSuites: Option[immutable.Seq[String]] = None,
  enabledProtocols:    Option[immutable.Seq[String]] = None,
  clientAuth:          Option[TLSClientAuth]         = None,
  sslParameters:       Option[SSLParameters]         = None) =
  new HttpsConnectionContext(sslContext, sslConfig, enabledCipherSuites, enabledProtocols, clientAuth, sslParameters)
```

除 `outgoingConnection`、`newHostConnectionPool` 和 `cachedHostConnectionPool` 外，[`akka.http.scaladsl.Http`](https://doc.akka.io/api/akka-http/10.0.10/akka/http/scaladsl/Http$.html) 也定义了 `outgoingConnectionHttps`、`newHostConnectionPoolHttps` 和 `cachedHostConnectionPoolHttps`。除了这些方法的连接都是加密的，其他方面与无 `Https` 后缀的对应方法完全一致。

`singleRequest` 和 `superPool` 方法根据请求的协议决定加密状态，即请求 *https* URI 的请求将被加密，而请求 *http* URI 的则不会。

所有 HTTPS 连接的加密配置，即 `HttpsContext` 由如下逻辑确定：

1. 若已配置可选参数 `httpsContext`，则其会包含全部配置信息（优先级最高，高于默认配置 `HttpsContext`）
2. 若未配置 `httpsContext`（默认未配置），则使用默认的客户端配置 `HttpsContext`（可使用 `setDefaultClientHttpsContext` 设置）
3. 若未（通过 `setDefaultClientHttpsContext`）设置 `HttpsContext`，则使用默认系统配置

常见场景一般是，若默认系统 TLS 配置无法满足需求，则自定义一个 `HttpsContext` 实例，并通过 `Http().setDefaultClientHttpsContext` 方法设置。之后可以直接使用 `outgoingConnectionHttps`、`newHostConnectionPoolHttps`、`cachedHostConnectionPoolHttps`、`superPool` 或 `singleRequest` 等方法，而无需提供 `httpsContext` 参数，这些方法将基于客户端配置 `HttpsContext` 对连接加密。

若未定义 `HttpsContext`，则使用 Java 默认的 TLS 设置。自定义 `HttpsContext` 可能会使 Https 客户端更加不安全，所以要清楚自己在做什么！

## SSL-Config

Akka HTTP 极度依赖 [Lightbend SSL-Config](http://typesafehub.github.io/ssl-config/)，并将大部分 SSL/TLS 相关配置委托给它处理。SSL-Config 专注于提供默认即安全的 SSLContext 及相关配置。

参考 [SSL-Config](http://typesafehub.github.io/ssl-config/) 文档获取更多细节。

## 详细配置与 workarounds

[SSL-Config](http://typesafehub.github.io/ssl-config/) 是 Lightbend 公司维护的库，旨在简化 SSL/TLS 的相关配置（相对于使用 JDK 提供的原声 SSL APIs）。

SSL-Config 所有可用的配置选项都可以通过 `akka.ssl-config` 设置。

>注意

>当连接 HTTPS 主机遇到问题时，强烈推荐阅读 ssl-config 配置指导。尤其 [添加证书到可信仓库](http://typesafehub.github.io/ssl-config/WSQuickStart.html#connecting-to-a-remote-server-over-https) 一节，事实证明其非常有用。

>{% em color="red" %}警告{% endem %}

>通过“宽松”设置可以禁用特定检查，然而我们 **强烈推荐** 通过适当配置 TLS（例如，将可信 key 添加到 keystore 中）来解决该类问题，而非粗暴禁止。

>若因为错误配置的（或古老的）服务器，必须禁用特定检查，则不要在全局范围禁用（`application.conf`），推荐只为已明确得知必须禁用的连接配置“宽松”设置（译者注：即禁用）。具体方法在随后章节讲述。

### 主机名验证

主机名验证证明 Akka HTTP 客户端连接的服务器确实是其目标服务器。若无该检查，则可能发生中间人攻击（译者注：参考 [Wikipedia 中间人攻击](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)），此时 *certificate* 中可能包含其他主机名，因此检查 *certificate* 中的主机名与连接的目标主机名是否一致非常重要。

默认 `HttpsContext` 会开启主机名验证，Akka HTTP 依赖 [Lightbend SSL-Config] 实现该特性。Java 7 之后，JDK 添加主机名验证功能（Akka HTTP 借助 JDK 实现主机名验证），在 Java 6，该验证借助 ssl-config 手动实现。

更多内容，推荐阅读该博客：[fixing hostname verification](https://tersesystems.com/2014/03/23/fixing-hostname-verification/)。

### 服务器名字指示（Server Name Indication SNI）

SNI 是 TLS 扩展，用于防止中间人攻击。SNI 规定客户端发送的虚拟域名字，并将之作为 TLS 握手的一部分。

参考 [RFC 6066](https://tools.ietf.org/html/rfc6066#page-6)。

### 禁用 TLS

>{% em color="red" %}警告{% endem %}

>强烈不推荐禁用任何 TLS 安全特性，但有时需要 workarounds。

>在禁用任何特性之前，考虑是否可以通过 TLS 配置满足需求，比如通过 [trusting a certificate](https://typesafehub.github.io/ssl-config/WSQuickStart.html) 或 [configuring the trusted cipher suites](https://typesafehub.github.io/ssl-config/CipherSuites.html)。ssl-config 文档也有类似说明：[LooseSSL - Please read this before turning anything off!](https://typesafehub.github.io/ssl-config/LooseSSL.html#please-read-this-before-turning-anything-off)。

>若确实需要禁用，推荐仅禁用特定连接，而非通过 `application.conf` 全局禁用。

下例禁用了给定连接的 SNI：

```scala
implicit val system = ActorSystem()
implicit val mat = ActorMaterializer()

// WARNING: disabling SNI is a very bad idea, please don't unless you have a very good reason to.
val badSslConfig = AkkaSSLConfig().mapSettings(s => s.withLoose(s.loose.withDisableSNI(true)))
val badCtx = Http().createClientHttpsContext(badSslConfig)
Http().outgoingConnectionHttps(unsafeHost, connectionContext = badCtx)
```

`badSslConfig` 是默认 `AkkaSSLConfig` 的副本，但配置修改为禁止 SNI，可以将其缓存起来，用于其他连接（一般不应该使用该特性）。
