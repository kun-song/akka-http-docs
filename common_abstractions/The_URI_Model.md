# URI 模型

Akka HTTP 有自己专用的 `Uri` 模型类，该类性能良好，且可用于其他 HTTP 模型的惯用法中。例如 `HttpRequest` 的目标 URI 将被解析为该类，其中所有字符都被转义，并施加了其他 URI 特有的语义。

## 解析 URI 字符串

我们遵循 [RFC 3986](https://tools.ietf.org/html/rfc3986#section-1.1.2) 实现 URI 解析规则。在解析 URI 字符串时，Akka HTTP 内部将创建一个 `Uri` 实例，其中包含解析出来的 URI 组成部分。

下例创建了一个 `Uri` 实例：

```scala
Uri("http://localhost")
```

下面例子包含更多有效的 URI 字符串，并展示了使用 `Uri.from()` 方法，`scheme`, `host`, `path` 和 `query` 作为参数，构建 `Uri` 实例的方式。

```scala
Uri("ftp://ftp.is.co.za/rfc/rfc1808.txt") shouldEqual
  Uri.from(scheme = "ftp", host = "ftp.is.co.za", path = "/rfc/rfc1808.txt")

Uri("http://www.ietf.org/rfc/rfc2396.txt") shouldEqual
  Uri.from(scheme = "http", host = "www.ietf.org", path = "/rfc/rfc2396.txt")

Uri("ldap://[2001:db8::7]/c=GB?objectClass?one") shouldEqual
  Uri.from(scheme = "ldap", host = "[2001:db8::7]", path = "/c=GB", queryString = Some("objectClass?one"))

Uri("mailto:John.Doe@example.com") shouldEqual
  Uri.from(scheme = "mailto", path = "John.Doe@example.com")

Uri("news:comp.infosystems.www.servers.unix") shouldEqual
  Uri.from(scheme = "news", path = "comp.infosystems.www.servers.unix")

Uri("tel:+1-816-555-1212") shouldEqual
  Uri.from(scheme = "tel", path = "+1-816-555-1212")

Uri("telnet://192.0.2.16:80/") shouldEqual
  Uri.from(scheme = "telnet", host = "192.0.2.16", port = 80, path = "/")

Uri("urn:oasis:names:specification:docbook:dtd:xml:4.1.2") shouldEqual
  Uri.from(scheme = "urn", path = "oasis:names:specification:docbook:dtd:xml:4.1.2")
```

对于 URI 各部分（比如 `scheme` `path` `query`）的准确定义，请参考 [RFC 3986](https://tools.ietf.org/html/rfc3986#section-1.1.2)，下面是基本概览：

```
  foo://example.com:8042/over/there?name=ferret#nose
  \_/   \______________/\_________/ \_________/ \__/
   |           |            |            |        |
scheme     authority       path        query   fragment
   |   _____________________|__
  / \ /                        \
  urn:example:animal:ferret:nose
```

对于 URI 中的特殊字符，使用 {% em %}percent encoding{% endem %} 处理，查看 [Query String in URI](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/common/uri-model.html#query-string-in-uri) 查看更多关于 percent encoding 的细节：

```scala
// don't double decode
Uri("%2520").path.head shouldEqual "%20"
Uri("/%2F%5C").path shouldEqual Path / """/\"""
```

>译者注：Percent-encoding 即为 URL encoding，上例中 `%` 被编码为 `%20`。

### 无效的 URI 字符串与 `IllegalUriException`

当传递无效的 URI 字符串给 `Uri()` 方法时，抛出 `IllegalUriException` 异常：

```scala
//illegal scheme
the[IllegalUriException] thrownBy Uri("foö:/a") shouldBe {
  IllegalUriException(
    "Illegal URI reference: Invalid input 'ö', expected scheme-char, 'EOI', '#', ':', '?', slashSegments or pchar (line 1, column 3)",
    "foö:/a\n" +
      "  ^")
}

// illegal userinfo
the[IllegalUriException] thrownBy Uri("http://user:ö@host") shouldBe {
  IllegalUriException(
    "Illegal URI reference: Invalid input 'ö', expected userinfo-char, pct-encoded, '@' or port (line 1, column 13)",
    "http://user:ö@host\n" +
      "            ^")
}

// illegal percent-encoding
the[IllegalUriException] thrownBy Uri("http://use%2G@host") shouldBe {
  IllegalUriException(
    "Illegal URI reference: Invalid input 'G', expected HEXDIG (line 1, column 13)",
    "http://use%2G@host\n" +
      "            ^")
}

// illegal path
the[IllegalUriException] thrownBy Uri("http://www.example.com/name with spaces/") shouldBe {
  IllegalUriException(
    "Illegal URI reference: Invalid input ' ', expected '/', 'EOI', '#', '?' or pchar (line 1, column 28)",
    "http://www.example.com/name with spaces/\n" +
      "                           ^")
}

// illegal path with control character
the[IllegalUriException] thrownBy Uri("http:///with\newline") shouldBe {
  IllegalUriException(
    "Illegal URI reference: Invalid input '\\n', expected '/', 'EOI', '#', '?' or pchar (line 1, column 13)",
    "http:///with\n" +
      "            ^")
}
```

### 提取 URI 组成元素的指令

想要使用指令提取 URI 元素，请参考如下链接：

* [extractUri](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/basic-directives/extractUri.html)
* [extractScheme](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/scheme-directives/extractScheme.html)
* [scheme](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/scheme-directives/scheme.html)
* [PathDirectives](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/path-directives/index.html)
* [ParameterDirectives](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/parameter-directives/index.html)

## 获取请求的原始 URI

有时需要获取进入的 URI 的 **原始值**，即没有经过解析、转义的值。虽然该场景很少，但总时不时出现。在 Akka HTTP 服务端中，若打开 `akka.http.server.raw-request-uri-header` 开关，则每个请求中都会添加一个 `Raw-Request-URI` 头部，其中包含请求 URI 的原始值。

## URI 中的查询字符串

虽然 URI 的任何部分都可以包含 {% em %}特殊字符{% endem %}，但相对而言，查询字符串（percent encoded）更容易包含特殊字符。

`Uri` 类的 `query()` 方法返回对应 URI 的查询字符串，其被建模为 `Query` 类的实例。当使用 URI 字符串创建 `Uri` 实例时吗，查询字符串以 **原始字符串** 形式存放，当调用 `query()` 方法时，查询字符串被解析、处理为 `Query` 实例。

下面展示了有效的查询字符串的解析，特别注意 percent-encoding 是如何使用的，以及特殊字符（比如 `+` `;`）是如何处理的：

>注意：`Query（）` 和 `Uri.query()` 方法的 `mode` 参数在 [Strict and Relaxed Mode](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/common/uri-model.html#strict-and-relaxed-mode) 一节介绍。

```scala
def strict(queryString: String): Query = Query(queryString, mode = Uri.ParsingMode.Strict)

//query component "a=b" is parsed into parameter name: "a", and value: "b"
strict("a=b") shouldEqual ("a", "b") +: Query.Empty

strict("") shouldEqual ("", "") +: Query.Empty
strict("a") shouldEqual ("a", "") +: Query.Empty
strict("a=") shouldEqual ("a", "") +: Query.Empty

strict("a=+") shouldEqual ("a", " ") +: Query.Empty //'+' is parsed to ' '
strict("a=%2B") shouldEqual ("a", "+") +: Query.Empty

strict("=a") shouldEqual ("", "a") +: Query.Empty
strict("a&") shouldEqual ("a", "") +: ("", "") +: Query.Empty
strict("a=%62") shouldEqual ("a", "b") +: Query.Empty

strict("a%3Db=c") shouldEqual ("a=b", "c") +: Query.Empty
strict("a%26b=c") shouldEqual ("a&b", "c") +: Query.Empty
strict("a%2Bb=c") shouldEqual ("a+b", "c") +: Query.Empty
strict("a%3Bb=c") shouldEqual ("a;b", "c") +: Query.Empty

strict("a=b%3Dc") shouldEqual ("a", "b=c") +: Query.Empty
strict("a=b%26c") shouldEqual ("a", "b&c") +: Query.Empty
strict("a=b%2Bc") shouldEqual ("a", "b+c") +: Query.Empty
strict("a=b%3Bc") shouldEqual ("a", "b;c") +: Query.Empty

strict("a+b=c") shouldEqual ("a b", "c") +: Query.Empty //'+' is parsed to ' '
strict("a=b+c") shouldEqual ("a", "b c") +: Query.Empty //'+' is parsed to ' '
```

注意：

```scala
Uri("http://localhost?a=b").query()
```

等价于：

```scala
Query("a=b")
```

[RFTC 3986 3.4 节](https://tools.ietf.org/html/rfc3986#section-3.4) 规定，允许部分特殊字符（比如 `/` 和 `?`）出现在查询字符串中，无法使用 `%` 进行转义。

>The characters slash (“/”) and question mark (“?”) may represent data within the query component.(RFTC 3986 3.4 节)

`/` 和 `?` 主要用于 URI 的查询参数中包含另一个 URI 的场景：

```scala
strict("a?b=c") shouldEqual ("a?b", "c") +: Query.Empty
strict("a/b=c") shouldEqual ("a/b", "c") +: Query.Empty

strict("a=b?c") shouldEqual ("a", "b?c") +: Query.Empty
strict("a=b/c") shouldEqual ("a", "b/c") +: Query.Empty
```

但其他特殊字符，若不使用其 percent-encoding 替代，则抛出 `IllegalUriException` 异常：

```scala
the[IllegalUriException] thrownBy strict("a^=b") shouldBe {
  IllegalUriException(
    "Illegal query: Invalid input '^', expected '+', '=', query-char, 'EOI', '&' or pct-encoded (line 1, column 2)",
    "a^=b\n" +
      " ^")
}
the[IllegalUriException] thrownBy strict("a;=b") shouldBe {
  IllegalUriException(
    "Illegal query: Invalid input ';', expected '+', '=', query-char, 'EOI', '&' or pct-encoded (line 1, column 2)",
    "a;=b\n" +
      " ^")
}
```

```scala
//double '=' in query string is invalid
the[IllegalUriException] thrownBy strict("a=b=c") shouldBe {
  IllegalUriException(
    "Illegal query: Invalid input '=', expected '+', query-char, 'EOI', '&' or pct-encoded (line 1, column 4)",
    "a=b=c\n" +
      "   ^")
}
//following '%', it should be percent encoding (HEXDIG), but "%b=" is not a valid percent encoding
the[IllegalUriException] thrownBy strict("a%b=c") shouldBe {
  IllegalUriException(
    "Illegal query: Invalid input '=', expected HEXDIG (line 1, column 4)",
    "a%b=c\n" +
      "   ^")
}
```

### Strict 模式与 Relaxed 模式

`Uri.query()` 方法和 `Query()` 都有一个 `mode` 参数，其值为 `Uri.ParsingMode.Strict` 或者 `Uri.ParsingMode.Relaxed`。两种模式对部分特殊字符的解析方式不同。

```scala
def relaxed(queryString: String): Query = Query(queryString, mode = Uri.ParsingMode.Relaxed)
```

在 `Strict` 模式下， 下面给两个例子都会抛出 `IllegalUriException` 异常：

```scala
the[IllegalUriException] thrownBy strict("a^=b") shouldBe {
  IllegalUriException(
    "Illegal query: Invalid input '^', expected '+', '=', query-char, 'EOI', '&' or pct-encoded (line 1, column 2)",
    "a^=b\n" +
      " ^")
}
the[IllegalUriException] thrownBy strict("a;=b") shouldBe {
  IllegalUriException(
    "Illegal query: Invalid input ';', expected '+', '=', query-char, 'EOI', '&' or pct-encoded (line 1, column 2)",
    "a;=b\n" +
      " ^")
}
```

但在 `Relaxed` 模式下，这些特殊字符将按原样输出：

```scala
relaxed("a^=b") shouldEqual ("a^", "b") +: Query.Empty
relaxed("a;=b") shouldEqual ("a;", "b") +: Query.Empty
relaxed("a=b=c") shouldEqual ("a", "b=c") +: Query.Empty
```

然而，即使在 `Relaxed` 模式下，依然有部分特殊字符需要使用 percent-encoding 编码：

```scala
//following '%', it should be percent encoding (HEXDIG), but "%b=" is not a valid percent encoding
//still invalid even in relaxed mode
the[IllegalUriException] thrownBy relaxed("a%b=c") shouldBe {
  IllegalUriException(
    "Illegal query: Invalid input '=', expected HEXDIG (line 1, column 4)",
    "a%b=c\n" +
      "   ^")
}
```

除了在 `mode` 参数中指定模式以外，还可以在配置文件中指定：

```scala
# Sets the strictness mode for parsing request target URIs.
    # The following values are defined:
    #
    # `strict`: RFC3986-compliant URIs are required,
    #     a 400 response is triggered on violations
    #
    # `relaxed`: all visible 7-Bit ASCII chars are allowed
    #
    uri-parsing-mode = strict
```

若要获取 URI 查询部分的原始形式，可借助 `Uri` 类的 `rawQueryString` 成员。

### 使用指令抽取查询参数

若要使用指令提取查询参数，请参考：

* [parameters](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/parameter-directives/parameters.html)
* [parameter](http://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/parameter-directives/parameter.html)
