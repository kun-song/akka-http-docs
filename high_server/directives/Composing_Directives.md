# 组合指令

>注意
>若遗漏指令之间的 `~` 字符，代码依然可以编译通过，但其行为会与预期不一致。原本 `~` 将仅生产一个表达式，遗漏后实际变为多个表达式，而且 **只有最后一个** 会生效。除 `~` 操作符外，也可以使用 `concat` 组合子，`concat(a, b, c)` 与 `a ~ b ~ c` 是等价的。

到目前为止，都是使用 **嵌套** 来组合指令，例如：

```scala
val route: Route =
  path("order" / IntNumber) { id =>
    get {
      complete {
        "Received GET request for order " + id
      }
    } ~
    put {
      complete {
        "Received PUT request for order " + id
      }
    }
  }

verify(route)
```

该例子中，`get` 与 `put` 指令使用 `~` 操作符连接，组合成一个更高层的路由，该路由将作为 `path` 指令的内部路由，为更清晰地表达该概念，将整个例子重新实现为：

```scala
def innerRoute(id: Int): Route =
  get {
    complete {
      "Received GET request for order " + id
    }
  } ~
  put {
    complete {
      "Received PUT request for order " + id
    }
  }

val route: Route = path("order" / IntNumber) {
  id => innerRoute(id)
}

verify(route)
```

从上面的例子中无法看到的是，指令并非实现为简单的方法，而是实现为类型为 `Directive` 的单独的对象，这有利于更加灵活地组合指令，例如，可以将 `|` 操作符用于指令：

```scala
val route =
  path("order" / IntNumber) { id =>
    (get | put) { ctx =>
      ctx.complete(s"Received ${ctx.request.method.name} request for order $id")
    }
  }

verify(root)
```

上面的例子使用了显式 `Route` 函数（记住 `Route` 仅仅是一个函数别名），最好不要使用：

```scala
val route =
  path("order" / IntNumber) { id =>
    (get | put) {
      extractMethod { m =>
        complete(s"Received ${m.name} request for order $id")
      }
    }
  }

verify(root)
```

若 `(get | put)` 片段在一个很大的路由结构中出现多次，则可以将其抽取出来：

```scala
val getOrput = get | put
val route =
  path("order" / IntNumber) { id =>
    getOrput {
      extractMethod { m =>
        complete(s"Received ${m.name} request for order $id")
      }
    }
  }

verify(root)
```
* 因为 `getOrput` 不接受任何参数，所以在这里可以使用 `val` 声明；

嵌套路由中也可以使用 `&` 操作符：

```scala
val getOrput = get | put
val route =
  (path("order" / IntNumber) & getOrput & extractMethod) { (id, m) =>
    complete(s"Received ${m.name} request for order $id")
  }

verify(route)
```

可以看到，当使用 `&` 操作符组合执行 *抽取* 操作的指令时，组合后的指令将简单的抽取其子指令的抽取内容。

这里依然可以进行抽象提取：

```scala
val orderGetOrPutWithMethod =
  path("order" / IntNumber) & (get | put) & extractMethod
val route =
  orderGetOrPutWithMethod{ (id, m) =>
    complete(s"Received ${m.name} request for order $id")
  }

verify(root)
```

这些例子中，使用了两个技巧：

* 使用 `|` 和 `&` 操作符组合指令
* 使用 `val` 提取指令

这两个技巧应用非常广泛，凡是接受内部路由的指令都可以使用。

注意，到目前为止的例子中，将多个指令 *压缩* 为一个指令并没有得到可读性、可维护更好的代码，事实上，第一个例子才是可读性最好的那个。

然而，这些例子仅仅是为了展示指令的灵活之处，应该根据具体应用选择合适的抽象层次。

## 使用 `concat` 组合子组合指令

除 `~` 操作符外，也可以使用 `concat` 组合子来组合指令，其中被组合的指令作为参数传递给 `concat`：

```scala
def innerRoute(id: Int): Route =
  concat(get {
      complete { "Received GET request for order " + id }
    },
    put {
      complete { "Received PUT request for order " + id }
    })

val route: Route = path("order" / IntNumber) { id => innerRoute(id) }

verify(route)
```
