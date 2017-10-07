# Server-Sent Events 支持

*Server-Sent Events* 是一个轻量级、[标准化](https://www.w3.org/TR/eventsource/) 的协议，支持服务端向客户端的消息推送。相对全双工的 WebSocket，SSE 仅支持服务端往客户端的单向通信。其优点为非常简单、仅依赖 HTTP 协议，且支持重试机制。

根据 SSE 标准，客户端向服务端请求 **事件流**，服务端响应的 `MediaType` 为 `text/event-stream`，使用 UTF-8 编码，并保持 {% em %}连接开放{% endem %}，以便向客户端发送事件。事件为结构化文本，包含不同字段，且以空行结束：

```scala
data: { "username": "John Doe" }
event: added
id: 42

data: another event
```

再（断开）重连后，客户端可通过 `Last-Event-ID` 头部告知服务端，之前接收到的最后一个事件。
