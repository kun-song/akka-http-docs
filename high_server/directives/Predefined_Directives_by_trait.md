# 预定义指令（以特质分类）

所有预定义的指令都可以特质分类，共同组成富有表现力的 `Directives` 特质。

## 过滤、提取请求

* [MethodDirective](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/method-directives/index.html)
  + 基于请求的 HTTP 方法过滤、提取
* [HeaderDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/header-directives/index.html)
  + 基于请求头过滤、提取
* [PathDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/path-directives/index.html)
  + 基于请求的 URI 路径过滤、提取
* [HostDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/host-directives/index.html)
  + 基于请求的目标主机过滤、提取
* [ParameterDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/parameter-directives/index.html)，[FormFieldDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/form-field-directives/index.html)
  + 基于请求的 query 参数或者表单字段过滤、提取
* [CodingDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/coding-directives/index.html)
  + 基于请求内容过滤、处理
* [Marshalling Directives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/marshalling-directives/index.html)
  + 提取请求实体
* [SchemeDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/scheme-directives/index.html)
  + 基于请求的 scheme 过滤、提取
* [SecurityDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/cookie-directives/index.html)
  + 处理请求的鉴权数据
* [CookieDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/cookie-directives/index.html)
  + 过滤、提取 cookies
* [BasicDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/basic-directives/index.html) 和 [MiscDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/misc-directives/index.html)
  + 处理请求的属性
* [FileUploadDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/file-upload-directives/index.html)
  + 处理文件上传

## 创建、转换响应

* [CacheConditionDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/cache-condition-directives/index.html)
  + 支持条件请求（`304 Not Modified` 响应）
* [CookieDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/cookie-directives/index.html)
  + 设置、修改、删除 cookies
* [CodingDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/coding-directives/index.html)
  + 压缩响应
* [FileAndResourceDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/file-and-resource-directives/index.html)
  + 从文件、资源中传递响应
* [RangeDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/range-directives/index.html)
  + 支持范围请求（`206 Partial Content` 响应）
* [RespondWithDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/respond-with-directives/index.html)
  + 修改响应属性
* [RouteDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/route-directives/index.html)
  + 完成、拒绝请求
* [BasicDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/basic-directives/index.html)，[MiscDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/misc-directives/index.html)
  + 处理、转换响应属性
* [TimeoutDirectives](https://doc.akka.io/docs/akka-http/10.0.10/scala/http/routing-dsl/directives/timeout-directives/index.html)
  + 配置请求超时时间，响应自动超时

## 预定义指令列表（以特质划分）

* BasicDirectives
* CacheConditionDirectives
* CodingDirectives
* CookieDirectives
* DebuggingDirectives
* ExecutionDirectives
* FileAndResourceDirectives
* FileUploadDirectives
* FormFieldDirectives
* FuturesDirectives
* HeaderDirectives
* HostDirectives
* Marshalling Directives
* MethodDirectives
* MiscDirectives
* ParameterDirectives
* PathDirectives
* RangeDirectives
* RespondWithDirectives
* RouteDirectives
* SchemeDirectives
* SecurityDirectives
* WebSocketDirectives
* TimeoutDirectives
