# XML 支持

Akka HTTP 的序列化、反序列化设置使其可以无缝支持特定的传输类型，比如 JSON，XML，甚至二进制编码。

Akka HTTP 通过 `akka-http-xml` 模块为 [Scala XML](https://github.com/scala/scala-xml) 提供开箱即用的支持。

## Scala XML 支持

`ScalaXMLSupport` 特质提供了 `FromEntityUnmarshaller[NodeSeq]` 和 `ToEntityMarshaller[NodeSeq]`，可以直接使用它们，也可以基于它们进行抽象。

为了使用 [Scala XML](https://github.com/scala/scala-xml) `NodeSeq` 支持 XML 序列化、反序列化，必须引入如下依赖：

```scala
"com.typesafe.akka" %% "akka-http-xml" % "10.0.10"
```

做完这些后，通过直接使用 `import akka.http.scaladsl.marshallers.xml.ScalaXmlSupport._` 或通过混入 `akka.http.scaladsl.marshallers.xml.ScalaXmlSupport` 特质，XML 与 `NodeSeq` 实例之间的序列化、反序列化即可透明进行。
