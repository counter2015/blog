# Scalatra in Action 4: 处理用户输入

title: Scalatra in Action 4. 处理用户输入
date: 2019-04-05 16:06:30
tags: [Scalatra, 读书笔记] 
categories: [技术]

------

## 请求的生存周期

生存周期这个词真是别扭，其实它的意思就是从一个请求的发起，到请求被处理，所经历的一系列过程的总和。

请求到达后，首先，Scalatra会检查请求的URL路径，查看是否有匹配的路由，一旦匹配上，就会依次执行如下操作：

- 参数会被提取存放至`params`的`map`中,以便之后的使用
- `before`的过滤操作被触发，可以在这里执行预处理
- 路由内的代码块被执行
- `after`的过滤操作被触发，可以在这里执行收尾
- 相应被写回并发送给请求端

如果没有匹配到任何路由，默认返回的是404，当然你也可以自定义想返回的任何东西，比如。。。

![](https://counter2015.com/picture/scalatra-4-1.jpg)

上一节我们了解了路由，这一节我们将搞明白路由进去后执行的操作。

## HTTP参数

如果你仔细观察过网页里的URL，你会发现各种奇怪的http参数，如：

```
https://www.baidu.com/s?wd=fsdfa&rsv_spt=1&rsv_iqid=0xd772c817000196a5&issp=1&f=8&rsv_bp=1&rsv_idx=2&ie=utf-8&tn=baiduhome_pg&rsv_enter=1&rsv_sug3=5&rsv_sug1=4&rsv_sug7=100&rsv_sug2=0&inputT=637&rsv_sug4=637
```

http参数是key-value对，key和value之间用分隔符隔开，如`wd=fsdfa&rsv_spt=1`,假如有这么一条请求`key1=hello&key2=world`被接受，你可以通过`params("key1")`来访问`hello`，通过`params("key2")`来访问`world`，当使用这种方法来获取请求中的值时，除非强制转换类型，都将被是作为`String`.

我们可以把参数分成三类

- 查询参数
- 路径参数
- 表单参数



**查询参数**

请求路径为 `http://localhost:8080/results?search_query=scalatra&op=scalatra`

```scala
get("/results") {
  val searchQuery = params("search_query")
  val originalQuery = params("oq")
  println(searchQuery) //"scalatra"
  println(originalQuery) //"scalatra"
}
```

**路径参数**

请求路径为 `http://localhost:8080/path/42`

```scala
get("/path/:something") {
    val something = params("something")
    <h1>{something}</h1>  // 42
}
```

**表单参数**

表单参数不会体现在URL中，而是作为HTTP请求的一部分，一个简单的构造办法是使用`curl`

```scala
post("/form") {
    val question = params("question")
    val answer = params("answer")
    <h1>{question}</h1>
    <h1>{answer}</h1>
}
```
```shell
$ curl --data-urlencode "question=world" --data-urlencode "answer=42" http://localhost:8080/form
ArrayBuffer(<h1>world</h1>, <h1>42</h1>)
```

## 多个参数

请求为：`GET http://localhost:8080/tagged?tag=tag1&tag=tag2`

```scala
get("/tagged") {
  println(params("tag")) // only "tag1"
  for(x <- multiParams("tag")) { 
    println(x) // "tag1" and "tag2"
  }
  val tags = multiParams("tag")
}
```

当然，如果输入了预料之外的参数，页面会报错

![](http://ww1.sinaimg.cn/large/005OIjE6ly1g1o9espgp7j30nq0epjs3.jpg)

为了避免在运行时由于未考虑到的输入，导致的`NullPointerException`，可以使用`Option`使得错误被提前预防。

```scala
// http://localhost:8080/tagged?xx=1&tag=22&tag=22&tag=abc => "22 22 abc"
// http://localhost:8080/tagged?xx=1                       => "Nothings"
get("/tagged") {
    val res = multiParams.get("tag") match {
      case Some(tags) => tags.mkString(" ")
      case None => "Nothings"
    }
    println(res)
  }
```



或者，也可以设定一个默认值

```scala
get("/tagged") {
  val search_query = params.getOrElse("search_query",
                                      halt(200, "Please provide a search query"))
  "You searched for '" + search_query + "'"
}
```

还可以使用`getAs[T]`将参数转换成对应类型

`params.getAs[Int]("price") // Option(42)`



## 自定义类型

要想实现getAs[自定义类型]，需要先满足以下几个条件

- 定义一个新的类型

  ```scala
  case class Name(firstName: String, lastName: String)
  ```

- 为了将`String`转换成`Name`，还需要写一个转换函数

  ```scala
  def toName(str: String) : Name =
      str.split(',').map(_.trim) match {
      case Array(lastName, firstName) => Name(lastName, firstName)
    }
  ```

- 需要定义一个type converter

  ```scala
  val stringToName: org.scalatra.util.conversion.TypeConverter[String, Name] = 
    safe { str =>toName(str)}
  ```

  在这里，`safe`代码块内，会捕获由错误的强制转换尝试导致的任何异常，并返回一个`Option`。

- 此时已经可以使用`params.getAs[Name]("name")(stringToName)`，但是可以通过scala的`implict val`来省略`(stringToName)`

  这是因为如果你试图将一个变量从一个类型强制转换为另一个类型，而编译器不知道该怎么做，那么它将在当前的作用域，搜寻是否有任何可以处理类型转换的隐式定义。这是scala的特性。<del>也是经常被用来装逼的特性</del>

  ```scala
  implicit val stringToName: org.scalatra.util.conversion.TypeConverter[String, Name] = 
    safe { str =>toName(str)}
  ```

综上所述，一个较为完整的代码如下

``` scala
import org.scalatra._
import org.scalatra.util.conversion.TypeConverter


class MyScalatraServlet extends ScalatraServlet {

  case class Name(firstName: String, lastName: String)

  def toName(str: String) : Name =
    str.split(',').map(_.trim) match {
    case Array(lastName, firstName) => Name(lastName, firstName)
  }

  implicit val stringToName: TypeConverter[String, Name] = safe { str =>toName(str)}

  post("/hackers") {
    val name = params.getAs[Name]("name").getOrElse(
      halt(BadRequest("Please provide a name")))
    val motto = params("motto")
    val birthYear = params.getAs[Int]("birth-year").getOrElse(
      halt(BadRequest("Please provide a year of birth.")))
    if (birthYear >= 1970) {
      println("Adding a hacker who was born within the Unix epoch.")
    } else {
      println("Adding a classical hacker.")
    }
    // Create a new hacker and redirect to /hackers/:slug
  }
}
```

## 过滤器

说是过滤器，同样很拗口，其实是Filters，具体的是指`before`和`after`的关键字。

```scala
// ???是scala的一个特性，可以用来代替"todo"而能顺利编译(其实啥都没做)，这里是做演示用
def ConnectDataBase = ???
def DisConnectDataBase = ???
  
// before filter在路由执行前运行
before() {
    // 设置contentType
    contentType = "text/html"
    ConnectDataBase
}

// after filter在执行(action)操作后运行
after() {
    DisConnectDataBase
}
```

可以指定路径设定过滤器，也可以设置过滤器的状态

```scala
before("/hackers", request.requestMethod == Post) {
  ConnectDataBase
}
```

## 其他用户输入

**Request headers**

有时您需要从传入请求中读取头。你可以用`request.getheader()方法。

例如，如果您想知道`text/html`是否是可接受的内容类型,对于给定的请求，可以通过执行以下操作进行检查：

```scala
request.getheader(“accept”).split(“，”).contains(“text/html”)
```



**[cookies](https://en.wikipedia.org/wiki/HTTP_cookie)**

可以通过`cookies.get`和`cookies.update`完成cookies的读写

```scala
get("/") {
  val previous = cookies.get("counter") match {
    case Some(v) => v.toInt
    case None => 0
  }

  cookies.update("counter", (previous+1).toString)
  <p>
    Hi, you have been on this page {previous} times already
  </p>
}
```



## Request Helpers

**Halting**

如果你想要在fliter或action内立刻停止操作，可以使用`halt()`

```scala
before(){
  if(params("name") == "Arthur") {
    halt(status = 403,
    headers = Map("X-Your-Mother-Was-A" -> "hamster",
    "X-And-Your-Father-Smelt-Of" -> "Elderberries"),
    body = <h1>No Way</h1>)
  }
}
```

![](http://ww1.sinaimg.cn/large/005OIjE6ly1g1odwnb62fj30iv04l74a.jpg)

**重定向**

```scala
get("/"){
  redirect("/someplace/else")
}
```

这段代码会返回http code 302，指向`/something/else`

`redirect`是`HaltException`的一个实现，如果不想触发这个异常，也可以用如下的方式来进行重定向

```scala
halt(status = 301, headers = Map("Location" -> "http://example.org/"))
```



## 小结

我们补充了前一章关于路由没讲完的部分，同时详细讲解action。

在阅读《Scalatra in Action》的过程中，发现书中部分代码有误，<del>当我去它的[github仓库](https://github.com/scalatra/scalatra-in-action)查看时发现坟头草都长3年高了，clone下来想提交个PR，结果还编译不过，issue里又没有解决办法，扎心了</del>。

![](http://ww1.sinaimg.cn/large/005OIjE6ly1g1ofappq1mj31h20a8jsw.jpg)

20190421更新：

时隔一个月，仓库的维护者答复说，[欢迎提交merge](<https://github.com/scalatra/scalatra/issues/895>)，于是立马fork了个用通过github网页直接改，<del>趁着人还没跑</del>立刻[提了过去](<https://github.com/scalatra/scalatra-website/pull/188>),达成生涯首个PR，<del>我也可以和别人吹牛逼说自己是给知名开源项目提过PR的`Contributor`了</del>

