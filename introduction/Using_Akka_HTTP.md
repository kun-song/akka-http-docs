# 使用 Akka HTTP

使用 `Akka HTTP` 需要包含以下依赖：

```scala
// For Akka 2.4.x or 2.5.x
"com.typesafe.akka" %% "akka-http" % "10.0.10"
// Only when running against Akka 2.5 explicitly depend on akka-streams in same version as akka-actor
"com.typesafe.akka" %% "akka-stream" % "2.5.4"
"com.typesafe.akka" %% "akka-actor"  % "2.5.4"
```

注意 `Akka HTTP` 包含两个模块：`akka-http` 和 `akka-http-core`。因为 `akka-http` 依赖 `akka-http-core`，所以不需要显示引入后者，但若仅使用底层抽象 API，可以只引入后者。最后确保 Scala 版本为 `2.11` 或 `2.12`。

更多基于 `Akka 2.5` 的信息，请查阅 [Compatibility Notes](http://doc.akka.io/docs/akka-http/current/scala/http/compatibility-guidelines.html#akka-http-10-0-x-with-akka-2-5-x)。
