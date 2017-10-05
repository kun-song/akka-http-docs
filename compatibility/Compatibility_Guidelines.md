# 兼容性指导

## 二进制兼容规则

Akka HTTP 的二进制兼容规则与 Akka 保持一致。简单讲 Akka HTTP 的版本由三部分组成：`major.minor.patch`，其中相同的 `major` 版本保证向后二进制兼容（除使用 `@ApiMayChange` `@InternalApi` `@DoNotInherit` 标记的 API，以及其他文档特殊说明的情况）。

更多规则细节请参考 [The @DoNotInherit and @ApiMayChange markers](https://doc.akka.io/docs/akka/2.4.19/common/binary-compatibility-rules.html#The_@DoNotInherit_and_@ApiMayChange_markers)

## 特殊版本讨论

当升级依赖库时可能会遇到如下异常：

```scala
Detected java.lang.NoSuchMethodError error, which MAY be caused by incompatible Akka versions on the classpath. Please note that a given Akka version MUST be the same across all modules of Akka that you are using, e.g. if you use akka-actor [2.5.3 (resolved from current classpath)] all other core Akka modules MUST be of the same version. External projects like Alpakka, Persistence plugins or Akka HTTP etc. have their own version numbers - please make sure you're using a compatible set of libraries.
```

### Akka HTTP 10.0.x 与 Akka 2.5.x

Akka HTTP 10.0.x 与 Akka `2.4.x` 和 `2.5.x` 两个版本都保持二进制兼容，但这需要构建系统依赖 `2.4` 系列的 Akka。根据实际情况不同，可能遇到以下场景：`akka-actor` 为 `2.5`，`akka-http` 为 `10.0.x`，此时 Akka HTTP 会引入 `2.4` 版本的 `akka-streams`（因为 Akka HTTP 默认依赖 Akka 2.4），最终破坏二进制兼容性，因为所有的 Akka 模块（Akka Streams 是 Akka 的一个模块，而 Akka HTTP 不是）必须保持相同版本。

为解决该问题，必须显式指明 Akka Streams 版本，使其与 Akka 保持一致：

```scala
val akkaVersion = "2.5.4"
val akkaHttpVersion = "10.0.10"
libraryDependencies += "com.typesafe.akka" %% "akka-http"   % akkaHttpVersion
libraryDependencies += "com.typesafe.akka" %% "akka-actor"  % akkaVersion
libraryDependencies += "com.typesafe.akka" %% "akka-stream" % akkaVersion
```

>译者注：Akka HTTP 构建系统依赖的是 Akka 2.4.x，从而实现与 2.4 2.5 两个版本的兼容性，而 Akka HTTP 中指向 Akka 的文档连接也全部是 2.4 版本。
