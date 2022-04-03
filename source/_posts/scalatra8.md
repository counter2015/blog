# Scalatra in Action 8.测试工程师要了杯啤酒


title: Scalatra in Action 8. 测试工程师要了杯啤酒
date: 2019-11-06 18:33:30
tags: [Scalatra, 读书笔记] 
categories: [技术]

------





## 前言

现在我们对Scalatra的基本功能都有了一定的了解，现在是时候了解测试了。

Scalatra的设计理念就是能够方便地使用你想要地插件，比如测试这里，你可以用ScalaTest或者Specs2或者任何你想要使用的测试框架。

下面将分别介绍这两者。



## Specs2

Scalatra是基于Java Servlet 构建的，尽管这使得它能够很好的支持像Jetty和Tomcat这样的服务器

但它同样给测试带来了以下的复杂度

- Servlet的主要方法的返回值都是Unit类型，这意味着我们需要检测其中对象的状态的变化
- API的函数过多，很难跟踪到具体位置，以3.0版本的`HttpServletRequest`为例，就有[87个](https://tomcat.apache.org/tomcat-7.0-doc/servletapi/javax/servlet/http/HttpServletRequestWrapper.html )(算上重载的函数)
- Servlet的规范限制了不能在任意时刻调用方法，所以很难模拟数据



由于以上原因，直接测试应用的Servlet层是相当困难的，Scalatra给出的解决方式是，通过DSL，调用HTTP客户端，与其内嵌的Servlet容器来通信。



### 引入Specs2依赖

```scala
libraryDependencies ++= Seq(
  "org.scalatra" %% "scalatra-specs2" % ScalatraVersion % "test"
)
```

上述代码添加至`build.sbt`的合适位置即可引入依赖，注意到这里最后多了`% "test"`，

这是声明这个依赖只能在测试时被访问，这样做的好处之一时避免在生产环境中引入不必要的依赖，增加复杂度



默认测试类的存放路径为`src/test/scala`([Maven 惯例](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html ))

下面我们以测试一个json结果为例，需要先引入对应的json依赖(json4s和Scalatra json集成组件)

```scala
libraryDependencies ++= Seq(
  "org.scalatra" %% "scalatra-json" % ScalatraVersion,
  "org.json4s" %% "json4s-jackson" % "3.3.0"
)
```

(尽管`json4s-jackson`已经有了[更新的版本](https://mvnrepository.com/artifact/org.json4s/json4s-jackson), 这里为了防止版本不兼容可能导致的问题，沿用了书上较旧的版本)

我们来看下面的代码

这是第一部分，控制主要逻辑，当用户访问`/foods/potatoes`时，返回一个json文本

```json
{
  "name":"potatoes",
  "fairTrade":true,
  "tags":[
    "vegetable",
    "tuber"]
}
```



```scala
package com.example.app

import org.json4s.DefaultFormats
import org.json4s.JsonDSL._
import org.scalatra.ScalatraServlet
import org.scalatra.json.JacksonJsonSupport

// src/main/scala/com.example.app/FoodServlet.scala
class FoodServlet extends ScalatraServlet with JacksonJsonSupport {

  implicit lazy val jsonFormats: DefaultFormats.type = DefaultFormats

  get("/foods/potatoes") {
    val productJson =
      ("name" -> "potatoes") ~
        ("fairTrade" -> true) ~
        ("tags" -> List("vegetable", "tuber"))
    productJson
  }
}
```

需要稍微修改一下程序的入口

```scala
import com.example.app.FoodServlet
import org.scalatra._
import javax.servlet.ServletContext

// src/main/scala/ScalatraBootstrap.scala 
class ScalatraBootstrap extends LifeCycle {
  override def init(context: ServletContext) {
    context.mount(new FoodServlet, "/*")
  }
}

```

这里可能会有点疑问，如果程序入口不叫`ScalatraBootstrap`行吗，当然可以，只要修改

`web.xml`,添加`context-param.param-value`就行了

详细可以参考[官方文档](http://scalatra.org/guides/2.6/deployment/configuration.html)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee 
  http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
  version="3.1">

  <context-param>
    <param-name>org.scalatra.LifeCycle</param-name>
    <param-value>org.yourdomain.project.MyScalatraBootstrap</param-value>
  </context-param>
  <listener>
    <listener-class>org.scalatra.servlet.ScalatraListener</listener-class>
  </listener>
</web-app>
```



有了这两段代码，我们的“web应用”（袖珍版）就能在浏览器上跑了。

现在是一个很简单的程序，但是如果以后要在这上面开发新的功能，如何保证引入的新代码不会破坏就有的功能呢？

这就需要写测试了。

有了测试，我们能更加放心地往上面开(hu)发(shi)

```scala
package com.example.app

import org.scalatra.test.specs2.ScalatraSpec
import org.specs2.matcher.MatchResult
import org.specs2.specification.core.SpecStructure

class FoodServletSpec extends ScalatraSpec {

  def potatoesOk: MatchResult[Any] =
    get("/foods/potatoes") {
      // Assert that the status of response was 200(OK)
      status must_== 200
    }

  // Specs2 syntax describes what will be asserted
  // using string to describe the test
  def is: SpecStructure =
    s2"""
         GET /foods/potatoes on FoodServlet
         should return status 200
      $potatoesOk
    """

  // Mounts the servlet to the root path so it can be called
  addServlet(classOf[FoodServlet], "/*")

}
```



这里面有一个奇怪的语法需要讲解下。

如果你熟悉Scala的字符串插值，如`s"my name is $name"`, 那么对接下来的部分会更容易理解。

`s2`这个前缀使用Scala自定义字符串插值的特性，来实现把测试的描述从代码中分离的功能。



在这段代码中，`is`不是一个字符串，而是会被转换成一个Spec2的数据结构`Fragments`.

这个数据结构用来具体描述测试样例。

详细的内容可以参考这篇[博客](http://etorreborre.blogspot.com/2013/05/the-latest-release-of-specs2-2.html)

上面的测试样例只是对比了HTTP的返回码，并没有考虑到其他的要素，下面的部分会针对这里做详细的展开。



### HTTP response 断言



HTTP response 可以看作由如下几部分组成：

- 状态码
- HTTP头
- 消息体

下面是更加详细的测试断言

```scala
package com.example.app

import org.scalatra.test.specs2.ScalatraSpec
import org.specs2.matcher._
import org.specs2.specification.core.SpecStructure

class FoodServletSpec extends ScalatraSpec {
  
  def is: SpecStructure =
    s2"""
         GET /foods/potatoes on FoodServlet
         should return status 200     $potatoesOk
         should be JSON               $potatoesJSON
         should contain name potatoes $potatoesName
    """

  addServlet(classOf[FoodServlet], "/*")

  val testURL = "/foods/potatoes"

  def potatoesOk: MatchResult[Any] =
    get(testURL) {
      status must_== 200
    }

  def potatoesJSON: MatchResult[String] = get(testURL) {
    header("Content-Type") must startWith("application/json;")
  }

  def potatoesName: MatchResult[String] = get(testURL) {
    body must contain("{\"name\":\"potatoes\"}")
  }

}
```

上面这部分的测试用来对比json字符串的方法是用string来做间接的比较，有没有办法直接比较json呢？



### 用JValues进行测试

我们之前也讲过关于json的使用，这里就不赘述了，直接给代码
```scala
import org.json4s._
import org.json4s.jackson.JsonMethods
class FoodServletSpec extends ScalatraSpec {
  def potatoesName: MatchResult[Any] = get(testURL) {
    val json = JsonMethods.parse(body)
    json \ "name" must_== JString("potatoes")
    json \ "fairTrade" must_== JBool(true)
    json \ "tags" must_== JArray(List(JString("vegetable"), JString("tuber")))
  }
}
```



> 懒出效率是程序员的美德



考虑一下，上面的代码是不是有重复的部分可以提取。

假如我们要测试多个不同的json数据，每次都要从HTTP response 中的body中将数据转换成json格式，这一步代码就会一直写。

一个简单的方法是把这部分提取出来。

```scala
import org.json4s.JValue
import org.json4s.jackson.JsonMethods

trait JsonBodySupport { 
  // Requires that children are Scalatra tests
  self: ScalatraTests =>
  def jsonBody: JValue = JsonMethods.parse(body)
}

class FoodServletSpec extends ScalatraSpec with JsonBodySupport {
  def potatoesName = get("/foods/potatoes") {
    jsonBody \ "name" must_== JString("potatos")
  }
}
```



这里用到了Scala的一个特性：[self-type](https://docs.scala-lang.org/tour/self-types.html)

我们来看这部分`self: ScalatraTests`,这里声明了，任何一个继承`JsonBoadySupport`的类，也一定是ScalatraTests的子类。

也就是说，下面这种定义，是行不通的

`class FoodServletSpec extends JsonBodySupport`

IDE会报错："illegal inheritance self-type does not conform to ScalaTests"

加上`extends ScalatraSpec`则不同，因为

```scala
trait ScalatraSpec extends SpecificationLike with BaseScalatraSpec
trait BaseScalatraSpec extends BeforeAfterAll with ScalatraTests
```

所以`ScalatraSpec`是`ScalaTests`的子类，可以正确地被定义。

那么下面的`JsonMethods.parse(body)`的"body"又是来自于哪里呢，这是因为

```scala
trait ScalatraTests extends EmbeddedJettyContainer with HttpComponentsClient 
trait HttpComponentsClient extends Client
trait Client extends ImplicitConversions {
  def body = response.body
}
```

经过如下改造后，我们新的测试代码如下：

```scala
package com.example.app

import org.json4s._
import org.json4s.jackson.JsonMethods
import org.scalatra.test.ScalatraTests
import org.scalatra.test.specs2.ScalatraSpec
import org.specs2.matcher._
import org.specs2.specification.core.SpecStructure


trait JsonBodySupport {
  sf: ScalatraTests =>
  def jsonBody: JValue = JsonMethods.parse(body)
}

class FoodServletSpec extends ScalatraSpec with JsonBodySupport {

  def is: SpecStructure =
    s2"""
         GET /foods/potatoes on FoodServlet
         should return status 200     $potatoesOk
         should be JSON               $potatoesJSON
         should contain name potatoes $potatoesName
    """

  addServlet(classOf[FoodServlet], "/*")

  val testURL = "/foods/potatoes"

  def potatoesOk: MatchResult[Any] =
    get(testURL) {
      status must_== 200
    }

  def potatoesJSON: MatchResult[String] = get(testURL) {
    header("Content-Type") must startWith("application/json;")
  }

  def potatoesName: MatchResult[Any] = get(testURL) {
    jsonBody \ "name" must_== JString("potatoes")
    jsonBody \ "fairTrade" must_== JBool(true)
    jsonBody \ "tags" must_== JArray(List(JString("vegetable"), JString("tuber")))
  }

}
```



### 运行测试

现在，我们可以进行简单的测试了

```shell
$ sbt test
```
shell会输出类似下面的内容
```plain
[info] FoodServletSpec
[info]          + GET /foods/potatoes on FoodServlet
[info]            should return status 200
[info]          + should be JSON
[info]          + should contain name potatoes
[info] Total for specification FoodServletSpec
[info] Finished in 850 ms
[info] 3 examples, 5 expectations, 0 failure, 0 error
20:00:21.390 [pool-26-thread-6] INFO  o.e.j.server.handler.ContextHandler - Stopped o.e.j.s.ServletContextHandler@6b12f7dd{/,file:///D:/IdeaProjects/scala/scalatra1/src/main/webapp/,UNAVAILABLE}
[info] MyScalatraServletTests:
[info] - GET / on MyScalatraServlet should return status 200
[info] ScalaTest
[info] Run completed in 2 seconds, 229 milliseconds.
[info] Total number of tests run: 1
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 1, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[info] Passed: Total 4, Failed 0, Errors 0, Passed 4
[success] Total time: 5 s, completed 2019-11-5 20:00:21
```

## 单元测试

单元测试有以下优点

- 易写
- 易部署
- 运行更快



下面通过业务代码来写一个单元测试

```scala
import com.example.app.{NukeLauncherServlet, RealNukeLauncher}
import org.scalatra._
import javax.servlet.ServletContext

// src/main/scala/ScalatraBootstrap.scala
class ScalatraBootstrap extends LifeCycle {
  override def init(context: ServletContext) {
    context.mount(new NukeLauncherServlet(RealNukeLauncher), "/nuke/*")
  }
}
```



```scala
package com.example.app

import org.scalatra.{Forbidden, ScalatraServlet}

// src/main/scala/coom.example.app/NukeLauncherServlet.scala
class NukeLauncherServlet(launcher: NukeLauncher )
  extends ScalatraServlet {
  val NuclearCode = "password123"
  post("/launch") {
    if (params("code") == NuclearCode)
      launcher.launch()
    else
      Forbidden()
  }
}

trait NukeLauncher {
  def launch(): Unit
}

object RealNukeLauncher extends NukeLauncher {
  def launch(): Unit = println("launched success.")
}

class StubNukeLauncher extends NukeLauncher {
  var isLaunched = false
  def launch(): Unit = isLaunched = true
}
```



我们应该注意到，在`StubNukeLauncher`中定义了一个**global var** `isLaunched`,这在scala中是一个不推荐的用法。



因为这会破坏线程安全和难以追踪其变化原因。

这里由于单元测试里不存在并发，同时，为了方便从外部直接修改，以进行测试（生产环境中需要连接数据库等，但测试时不应该直接连生产数据库，而是使用模拟数据），所以使用了`var`来定义变量。

```scala
package com.example.app

import org.scalatra.test.specs2.MutableScalatraSpec
import org.specs2.mutable.After

// src/test/scala/com.example.app/NukeLanucherSpec.scala
class NukeLauncherSpec extends MutableScalatraSpec with After{

  // Runs the tests sequentially because the stub is stateful
  sequential

  val stubLauncher = new StubNukeLauncher
  addServlet(new NukeLauncherServlet(stubLauncher), "/*")

  // Cleans up the state between test runs
  def after: Unit = stubLauncher.isLaunched = false

  // Factor out common logic
  def launch[A](code: String)(f: => A): A =
    post("/launch", "code" -> code)(f)

  "The wrong pass code" should {
    "respond with forbidden" in {
      launch("wrong") {
        status must_== 403
      }
    }
    
    "not launch the nukes" in {
      launch("wrong") {
        stubLauncher.isLaunched must_== false
      }
    }
  }
  
  "The right pass code" should {
    "launch the nukes" in {
      launch("password123") {
        stubLauncher.isLaunched must_== true
      }
    }
  }
}
```

值得注意的是Spec2是默认用并行来执行测试的，这是为了节省测试时间，但是在这个测试时，如果进行并行操作，由于不同的POST请求在同时修改可变变量`isLaunched`，这有可能会导致测试失败。

所以需要把测试声明成并行的`sequential`



## ScalaTest

ScalaTest和Specs2也是常见的Scala测试框架，两者有很多相似的语法。



### 引入ScalaTest依赖

```scala
libraryDependencies ++= Seq(
  "org.scalatra" %% "scalatra-scalatest" % ScalatraVersion % "test"
)
```

测试代码如下

```scala
package com.example.app

import org.json4s.JString
import org.scalatra.test.scalatest.ScalatraWordSpec

class FoodServletTest extends ScalatraWordSpec with JsonBodySupport {

  addServlet(classOf[FoodServlet], "/*")
  "GET /foods/potatoes on FoodServlet" must {
    "return status 200" in {
      get("/foods/potatoes") {
        status should equal(200)
      }
    }
    "be JSON" in {
      get("/foods/potatoes") {
        header("Content-Type") should startWith("application/json;")
      }
    }
    "should have name potatoes" in {
      get("/foods/potatoes") {
        jsonBody \ "name" should equal(JString("potatoes"))
      }
    }
  }
}
```

运行测试和Specs2类似，这里就不赘述了。