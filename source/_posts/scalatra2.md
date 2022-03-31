# Scalatra in Action 2. Scalatra,嘎嘣脆！

title: Scalatra in Action 2. Scalatra,嘎嘣脆！
date: 2019-02-10 18:32:30
tags: [Scalatra, 读书笔记] 
categories: [技术]



------

## 前言

本系列文章是在阅读《Scalatray in Action》时的学习记录，会参杂个人见解，欢迎勘误抓虫。

{% spoiler 标题创意来源于《python网络数据采集》"再端一碗Beautiful Soup" %}



## 下载模板

```shell
# 需要先安装giter8
$ g8 scalatra/scalatra-sbt.g8
# Starting sbt 0.13.13, Giter8 can be called from sbt’s "new" command as follows:
# 当然你也可以直接使用sbt new来创建
$  sbt new scalatra/scalatra-sbt.g8
...
organization [com.example]: 
name [My Scalatra Web App]: 
version [0.1.0-SNAPSHOT]: 
servlet_name [MyScalatraServlet]: 
package [com.example.app]: 
scala_version [2.12.6]: 
sbt_version [1.2.1]: 
scalatra_version [2.6.4]:  
...
```

这里会提示输入一些参数，直接一路回车就好了。

```shell
# 目录结构如下
$ tree
.
├── README.md
├── build.sbt
├── project
│   ├── build.properties
│   └── plugins.sbt
└── src
    ├── main
    │   ├── resources
    │   │   └── logback.xml
    │   ├── scala
    │   │   ├── ScalatraBootstrap.scala
    │   │   └── com
    │   │       └── example
    │   │           └── app
    │   │               └── MyScalatraServlet.scala
    │   ├── twirl
    │   │   ├── layouts
    │   │   │   └── default.scala.html
    │   │   └── views
    │   │       └── hello.scala.html
    │   └── webapp
    │       └── WEB-INF
    │           └── web.xml
    └── test
        └── scala
            └── com
                └── example
                    └── app
                        └── MyScalatraServletTests.scala

```



这里可以稍微解释下目录结构

- build.sbt scala项目的配置文件
- project 存放sbt版本信息和sbt插件信息
- src 这之后的目录结构参考[maven默认目录结构](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)



我们可以从sbt控制台启动该web程序

```shell
$ sbt
sbt:My Scalatra Web App> ~jetty:start


```

![](https://counter2015.com/picture/scalatra-1-1.jpg)

这个页面其实就是我们在[Hello world项目](http://counter2015.com/2019/01/04/scalatra1/)中的结果

## Scalatra中到底有啥



**Scalatra的核心只是一种路由HTTP请求的方式，以便在服务器上执行代码块**

在这之外的所有其他功能都是通过外部工具包来实现的，下面这张图可以展示一个直观的联系。

![](https://counter2015.com/picture/scalatra-2-1.jpg)



*▲ 图片来源：scalatra in action*

通过giter8模板自动生成的scalatra项目会带有

- `scalatra-scalatest`测试组件
- `logback-classic`日志记录组件
- `jetty-webapp`服务器



其中`MyScalatraServlet.scala`可以近似地看作为程序入口

```scala
package com.example.app

import org.scalatra._

class MyScalatraServlet extends ScalatraServlet {

  get("/") {
    views.html.hello()
  }

}
```

很简单对吧，不过就是这个`views.html.hello`有点不太好理解。

我们回到之前的程序的目录结构图

```
├── README.md
├── build.sbt
├── project
│   ├── build.properties
│   └── plugins.sbt
└── src
    ├── main
    │   ├── resources
    │   │   └── logback.xml
    │   ├── scala
    │   │   ├── ScalatraBootstrap.scala
    │   │   └── com
    │   │       └── example
    │   │           └── app
    │   │               └── MyScalatraServlet.scala
    │   ├── twirl
    │   │   ├── layouts
    │   │   │   └── default.scala.html
    │   │   └── views
    │   │       └── hello.scala.html
    │   └── webapp
    │       └── WEB-INF
    │           └── web.xml
    └── test
        └── scala
            └── com
                └── example
                    └── app
                        └── MyScalatraServletTests.scala
```



这个目录结构其实和典型的[maven标准目录结构](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)很像

稍有不同的地方在于

- `project/`  sbt版本信息，sbt插件
- `twirl/` play的web框架模板组件

需要简单说明下`twirl`的语法规则，参考[官方文档](https://www.playframework.com/documentation/2.6.x/ScalaTemplates)

### @ In Twirl

`@`在twirl中作为一个特殊的标识符，标志着动态代码的开始，代码的结尾不需要显示的标识（因为会自动检测）

如：

Hello @customer.name!
       ^^^^^^^^^^^^^
       Dynamic code



Hello @(customer.firstName + customer.lastName)!
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                    Dynamic Code



Hello @{val name = customer.firstName + customer.lastName; name}!
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^



My email is bob@@example.com



**模板参数**

模板类似于函数，因此它需要参数，这些参数必须在模板文件的顶部声明：

```html
@(customer: Customer, orders: List[Order])
//You can also use default values for parameters:

@(title: String = "Home")
//Or even several parameter groups:

@(title: String)(body: Html)
```



**循环**

```html
<ul>
@for(p <- products) {
  <li>@p.name ($@p.price)</li>
}
</ul>
```



Note: 由于<del>愚蠢的</del>编辑器，需要手动保证`{`和`for`在同一行以确保动态代码能正确生效。

> 关键字和动态代码中间也不能有空格，要是写成`@for (p <- products)`也会报编译错误

更多例子可以参照上面给出的官方文档的链接。



回到之前的话题：

`views.html.hello`到底是什么东西？

如果你在 IDEA中打开项目（尚未编译运行过）的话，你会发现`view`会被标红，但是当你从sbt运行时，它就好了。



```
   │   ├── twirl
    │   │   ├── layouts
    │   │   │   └── default.scala.html
    │   │   └── views
    │   │       └── hello.scala.html
```



这是因为在twirl框架中，`views/Application/index.scala.html`的模板文件会被编译成拥有`apply`方法的`views.html.Application.index`的类，在这个项目中，`views/hello.scala.html`生成的类`view.html.hello`就在`target/scala-2.12/twirl/main/views/html/hello.template.scala`

```scala

package views.html

import _root_.play.twirl.api.TwirlFeatureImports._
import _root_.play.twirl.api.TwirlHelperImports._
import _root_.play.twirl.api.Html
import _root_.play.twirl.api.JavaScript
import _root_.play.twirl.api.Txt
import _root_.play.twirl.api.Xml

object hello extends _root_.play.twirl.api.BaseScalaTemplate[play.twirl.api.HtmlFormat.Appendable,_root_.play.twirl.api.Format[play.twirl.api.HtmlFormat.Appendable]](play.twirl.api.HtmlFormat) with _root_.play.twirl.api.Template0[play.twirl.api.HtmlFormat.Appendable] {

  /**/
  def apply/*1.2*/():play.twirl.api.HtmlFormat.Appendable = {
    _display_ {
      {


Seq[Any](format.raw/*1.4*/("""
"""),_display_(/*2.2*/layouts/*2.9*/.html.default("Scalatra: a tiny, Sinatra-like web framework for Scala", "Welcome to Scalatra")/*2.103*/{_display_(Seq[Any](format.raw/*2.104*/("""
  """),format.raw/*3.3*/("""<p>Hello, Twirl!</p>
""")))}))
      }
    }
  }

  def render(): play.twirl.api.HtmlFormat.Appendable = apply()

  def f:(() => play.twirl.api.HtmlFormat.Appendable) = () => apply()

  def ref: this.type = this

}


              /*
                  -- GENERATED --
                  DATE: Wed Feb 13 14:24:12 GMT+08:00 2019
                  SOURCE: D:/IdeaProjects/scalatra-study/tmp/my-scalatra-web-app/src/main/twirl/views/hello.scala.html
                  HASH: 6a5309d0d842c7b54c83aa665c985afefdc12c70
                  MATRIX: 559->1|655->3|682->5|696->12|799->106|838->107|867->110
                  LINES: 14->1|19->1|20->2|20->2|20->2|20->2|21->3
                  -- GENERATED --
              */
          
```



可以看到，代码中`/*2.9*/`位置又调用了`layouts.html.default`

```scala
@(title: String, headline: String)(body: Html)
<html>
  <head>
    <title>@title</title>
  </head>
  <body>
    <h1>@headline</h1>
    @body
  </body>
</html>
```

生成的对应类`target/scala-2.12/twirl/main/views/html/default.template.scala`

```scala

package layouts.html

import _root_.play.twirl.api.TwirlFeatureImports._
import _root_.play.twirl.api.TwirlHelperImports._
import _root_.play.twirl.api.Html
import _root_.play.twirl.api.JavaScript
import _root_.play.twirl.api.Txt
import _root_.play.twirl.api.Xml

object default extends _root_.play.twirl.api.BaseScalaTemplate[play.twirl.api.HtmlFormat.Appendable,_root_.play.twirl.api.Format[play.twirl.api.HtmlFormat.Appendable]](play.twirl.api.HtmlFormat) with _root_.play.twirl.api.Template3[String,String,Html,play.twirl.api.HtmlFormat.Appendable] {

  /**/
  def apply/*1.2*/(title: String, headline: String)(body: Html):play.twirl.api.HtmlFormat.Appendable = {
    _display_ {
      {


Seq[Any](format.raw/*1.47*/("""
"""),format.raw/*2.1*/("""<html>
  <head>
    <title>"""),_display_(/*4.13*/title),format.raw/*4.18*/("""</title>
  </head>
  <body>
    <h1>"""),_display_(/*7.10*/headline),format.raw/*7.18*/("""</h1>
    """),_display_(/*8.6*/body),format.raw/*8.10*/("""
  """),format.raw/*9.3*/("""</body>
</html>"""))
      }
    }
  }

  def render(title:String,headline:String,body:Html): play.twirl.api.HtmlFormat.Appendable = apply(title,headline)(body)

  def f:((String,String) => (Html) => play.twirl.api.HtmlFormat.Appendable) = (title,headline) => (body) => apply(title,headline)(body)

  def ref: this.type = this

}


              /*
                  -- GENERATED --
                  DATE: Wed Feb 13 14:24:12 GMT+08:00 2019
                  SOURCE: D:/IdeaProjects/scalatra-study/tmp/my-scalatra-web-app/src/main/twirl/layouts/default.scala.html
                  HASH: ab59b4dddbafceb665de3cb0f7ce9f2c8dc5df51
                  MATRIX: 582->1|722->46|749->47|803->75|828->80|891->117|919->125|955->136|979->140|1008->143
                  LINES: 14->1|19->1|20->2|22->4|22->4|25->7|25->7|26->8|26->8|27->9
                  -- GENERATED --
              */
          
```



这个又长又乱的代码就不再仔细看了，<del>有兴趣可以慢慢研究</del>

现在我们可以大致的标一下程序运行的过程，可能不太严谨，只是为了有一个直观的认识。

1.  `ScalatraBootstrap`初始化，调用`MyScalatraServlet`
2.  `MyScalatraServlet`处理路由请求，对根目录下的请求转发到`views.html.hello`处理
3.  `views.html.hello`传递两个参数，作为title和headline，同时传入一个行内html代码块
4.  `default.template.scala`生成具体的html代码



如此一来，我们解决了第一章的大部分问题（除了启动流程还是有些模糊）

其实上面这一大坨等效于

```scala
package com.example.app

import org.scalatra._

class MyScalatraServlet extends ScalatraServlet {

  get("/") {
    //views.html.hello()
    <html>
      <head>
        <title>Scalatra: a tiny, Sinatra-like web framework for Scala</title>
      </head>
      <body>
        <h1>Welcome to Scalatra</h1>
        <p>Hello, Twirl!</p>
      </body>
    </html>
  }

}
```

{% spoiler 可见某些时候框架会***帮助***我们把问题搞得更复杂 %}

## 再咬上一口scalatra

上面这种写法虽然更加简单，但是也引入一个问题——我们无法对数据进行抽象，说到底，还是需要定义一个数据结构。

为了简单起见，我们可以直接在`MyScalatraServlet.scala`中添加代码(为什么加这样的数据类型当然是参照scala in action 偷懒呀)

```scala
case class Page(slug:String, title:String, summary:String, body: String)
object PageDao {
  val page1 = Page("bacon-ipsum",
    "Bacon ipsum dolor sit amet hamburger",
  """Shankle pancetta turkey ullamco exercitation laborum ut
      officia corned beef voluptate.""",
  """Fugiat mollit, spare ribs pork belly flank voluptate ground
      round do sunt laboris jowl. Meatloaf excepteur hamburger pork
      chop fatback drumstick frankfurter pork aliqua.
      Pork belly meatball meatloaf labore. Exercitation commodo nisi
      shank, beef drumstick duis. Venison eu shankle sunt commodo short
      loin dolore chicken prosciutto beef swine elit quis beef ribs.
      Short ribs enim shankle ribeye andouille bresaola corned beef
      jowl ut beef.Tempor do boudin, pariatur nisi biltong id elit
      dolore non sunt proident sed. Boudin consectetur jowl ut dolor
      sunt consequat tempor pork chop capicola pastrami mollit short
      loin.""")
  val page2 = Page("veggie-ipsum",
    "Arugula prairie turnip desert raisin sierra leone",
    """Veggies sunt bona vobis, proinde vos postulo esse magis napa
      cabbage beetroot dandelion radicchio.""",
    """Brussels sprout mustard salad jícama grape nori chickpea
      dulse tatsoi. Maize broccoli rabe collard greens jícama wattle
      seed nori garbanzo epazote coriander mustard.""")
  val pages = List(page1, page2)
}
```

关于`case calss` 可以参阅[官方文档](https://docs.scala-lang.org/tour/case-classes.html)和[另一篇文档](https://www.scala-exercises.org/scala_tutorial/classes_vs_case_classes)

`object`的话目前需要知道它是**单例**就行了

> Note: 实际使用过程中，当然不推荐把数据直接写到代码里，最好是存放到数据库中。



现在我们已经”存储“了一些页面，现在的问题是如何访问这些页面，还记得之前说的scalatra的核心是什么吗？

> At its core, Scalatra is nothing more than a way of routing HTTP requests in order to exe-
> cute blocks of code on the server. 									---《Scalatra in Action》

继续添加以下代码

```scala
get("/pages/:slug") {
    // Adds a content type on the response
    contentType = "text/html"
    
    //Calls the find method on your list of pages, and checks whether any page’s slug matches the incoming route parameter
    PageDao.pages find (_.slug == params("slug")) match {
      // If a page with a matching slug is found, displays the page’s title
      case Some(page) => page.title
        
      // If no page with a matching slug is found, stops and displays an erro
      case None => halt(404, "not found")
    }
  }
```

关于scalatra的路由语法可以参考[官方文档](http://scalatra.org//guides/2.6/http/routes.html)

简单来说http://localhost:8080/pages/anything-at-all 会触发这段代码,`params("slug")`的值为anything-at-all

这里发生了很多事情。路由的操作使用find方法scala的`List`类。在本例中它被用作通配符模式。

pagedao.pages列表中的每个页面对象实例都迭代发送到`match`函数，如果页的slug与参数（“slug”）匹配，则匹配块将生成那页的标题。如果找不到匹配页，则match块调用halt并返回HTTP 404状态代码。

听起来很复杂吗，不如直接运行看看吧。



<details>
  <summary>完整代码如下: </summary>
  <p>// MyScalatraServlet.scala</p>
  <pre><code> 
package com.example.app
import org.scalatra._
class MyScalatraServlet extends ScalatraServlet {
  get("/pages/:slug") {
    contentType = "text/html"
    PageDao.pages find (_.slug == params("slug")) match {
      case Some(page) => page.title
      case None => halt(404, "not found")
    }
  }
}
case class Page(slug:String, title:String, summary:String, body: String)
object PageDao {
  val page1 = Page("bacon-ipsum",
    "Bacon ipsum dolor sit amet hamburger",
    """Shankle pancetta turkey ullamco exercitation laborum ut
      officia corned beef voluptate.""",
    """Fugiat mollit, spare ribs pork belly flank voluptate ground
      round do sunt laboris jowl. Meatloaf excepteur hamburger pork
      chop fatback drumstick frankfurter pork aliqua.
      Pork belly meatball meatloaf labore. Exercitation commodo nisi
      shank, beef drumstick duis. Venison eu shankle sunt commodo short
      loin dolore chicken prosciutto beef swine elit quis beef ribs.
      Short ribs enim shankle ribeye andouille bresaola corned beef
      jowl ut beef.Tempor do boudin, pariatur nisi biltong id elit
      dolore non sunt proident sed. Boudin consectetur jowl ut dolor
      sunt consequat tempor pork chop capicola pastrami mollit short
      loin.""")
  val page2 = Page("veggie-ipsum",
    "Arugula prairie turnip desert raisin sierra leone",
    """Veggies sunt bona vobis, proinde vos postulo esse magis napa
      cabbage beetroot dandelion radicchio.""",
    """Brussels sprout mustard salad jícama grape nori chickpea
      dulse tatsoi. Maize broccoli rabe collard greens jícama wattle
      seed nori garbanzo epazote coriander mustard.""")
  val pages = List(page1, page2)
}
  </code>  </pre>
</details>


下面是运行结果

![](https://counter2015.com/picture/scalatra-2-2.jpg)





## 小结

我们简单调试了scalatra，但我们的问题也越来越多

- 如何单独部署写好的WEB项目
- 如何调整布局
- Scalatra的各种组件有哪些适用范围
- 如何测试

