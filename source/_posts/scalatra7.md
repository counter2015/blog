

# Scalatra in Action 7: 服务端模板

title: Scalatra in Action 7. 服务端模板
date: 2019-08-26 19:47:30
tags: [Scalatra, 读书笔记] 
categories: [技术]

------







## 选择模板引擎

### 网站

网站的架构如下图所示

![](<https://counter2015.com/picture/scalatra-7-1.png>)

▲ 图片来源：scalatra in action

1. 浏览器向scalatra发出请求。

2.  scalatra查询数据库。

3. 数据库以结果集响应。

4.  scalatra使用结果集调用scalate模板。

5.  scalate返回HTML视图。

6. scalatra将HTML视图返回给客户端

### Web APIs

![](<https://counter2015.com/picture/scalatra-7-2.png>)

这种架构下，基本上很少使用html模板。模板编程并不是必须的，比如restful API就可以不用模板。

## Scalate

Scalate是一个服务端模板框架，它有以下特性

强类型模板

传递自定义绑定的能力

自动模板重新加载和缓存

带跟踪输出和行号的错误信息

首先，我们在项目中添加依赖

```scala
"org.scalatra" %% "scalatra-scalate" % ScalatraVersion
```



下面是一段简单的有关Scalate的代码示例

```scala
package com.example.app

import org.scalatra.ScalatraServlet
import org.scalatra.scalate.ScalateSupport

// Mixes in the Scalatra-Scalate integration
trait MyScalatraWebAppStack extends ScalatraServlet with ScalateSupport {
  notFound {
    // Removes the content type in case it was set through an action integration
    contentType = null

    // Invokes the template
    findTemplate(requestPath).map { path =>
      contentType = "text/html"

      // If no explicit route is mapped, falls through to the not-found handle
      layoutTemplate(path)
    }.orElse(serveStaticResource) // If neither a route matches nor a template is found, looks for a static resource
      .getOrElse(resourceNotFound) // If it falls through to here, admits defeat

  }
}
```

### 目录结构

一个典型的Scalate结构如下

```plain
src/main/webapp
	js
	css
	WEB-INF
			layouts
				default.jade
			views
				hello-scalate.jade
```



`WEB-INF`文件夹在这里不太直观，其实这是java servlet标准的一个约定。比如，`GET`请求访问网站静态资源时，从路由`/images/banner.png`会访问`src/main/webapp/images/banner.png`



WEB-INF目录是特殊的，因为它可以屏蔽直接访问。您不希望服务在静态资源里找到scaml模板，因为它将呈现模板源而不是执行模板。除非显式调用模板，否则模板不可访问，因为它们位于特殊目录中。

## 使用模板来构建网页

### [Scalate](http://scalate.github.io/scalate/)

首先，让我们在`src/main/webapp/web-inf/views/greater.scaml`中创建视图。

```
-# This is a comment
```

以上是`scaml`中的注释格式，下面是具体代码

```scaml
-# Specifies the HTML doctype (here, HTML 5)
!!! 5
-# Declares an attribute, named `whom`, of type String
-@ val whom: String
-# Declares a second attribute, named `lucky`, of type Int
-@ val lucky: List[Int]
%html
  %head
    %link(type="text/css" href="/css/style.css" rel="stylesheet")
    -# use `whom`
    %title Hello, #{whom}
  %body
    %h1 Congratulations
    %p You've created your first Scalate view, #{whom}.
    %p Your lucky numbers are:
    %ul
      -# Loops with typical Scala code
      - for (number <- lucky)
        %li #{number}
```

解释参见代码中的注释部分。

### 排版

渲染视图时，Scalate会先到`/WEB-INF/layouts/default.{dialect}`寻找文件。这里`dialect`指的是某种支持的文件类型，包括

- scaml
- ssp
- jade
- mustache

![](<https://counter2015.com/picture/scalatra-7-3.png>)



下面是`default.scaml`的代码

```scaml
!!! 5
-@ val body: String
-@ val title: String
%html
  %head
    %link(type="text/css" href="/css/style.css" rel="stylesheet")
    -# renders the title
    %title= title
  %body
    -# renders the body
    != body
```

- `%title= title`创建一个HTML元素，是`%title #{title}`的缩写。
- !=用来渲染原始的内容，如"<"就会被渲染成“&lt”.



在这个基础上，我们编写下面的视图

```scaml
-@ val whom: String
-@ val lucky: List[Int]
- attributes("title") = "Hello, "+whom

%h1 Congratulations
%p You've created your first Scalate view, #{whom}.
%p Your lucky numbers are:
%ul
  - for (number <- lucky)
    %li #{number}
```



这段代码中最需要关注的是这一部分

`- attributes("title") = "Hello, "+whom`



布局需要标题(在`layout`文件夹中的变量`title`)。该行允许视图模板提供布局上属性的值。

你也许会觉得奇怪，同样是变量名称，为什么`body`就不需要赋值了呢？

这是因为`body`取的值是上面这个视图代码的整个的输出结果，这个是特例（数一下现在我们遇到过多少个特例/约定了），会按字符串个格式传入。



和`body`一样，`layout`也是一个特殊的属性。

目前为止，我们讲到了页面排版默认使用的配置文件，但是经常会有这么一个场景：对于登录的用户和未登录的访客，他们看到的东西应该是不一样的。为了实现这个效果，Scalate当然要支持执行`layout`配置文件

```plain
- attributes("layout") = "/WEB-INF/layouts/guest.scaml"
%h1 Welcome, guest
% This page will use the guest layout.
```



> Notes: 你可以将`layout`设置成空字符串“”来完全抑制布局这一功能





### 调用模板



```scala
package com.example.app

class GreeterServlet extends MyScalatraWebAppStack {
  get("/greet/:whom") {
    val lucky =
      for (i <- (1 to 5).toList)
        yield util.Random.nextInt(48) + 1
    layoutTemplate("greeter.html",
      "whom" -> params("whom"),
      "lucky" -> lucky)
  }
}
```



这里会搜寻模板，按照如下顺序

1. `/WEB-INF/views/greeter.html`
2. `/WEB-INF/views/greeter.html.scaml`
3. `/WEB-INF/views/greeter.scaml`
4. `/WEB-INF/views/greeter.html/index`

(无力吐槽，约定的东西太多了)

最后能跑的demo代码清单如下

`src/main/scala/ScalatraBootstrap.scala`

```scala
import com.example.app.{GreeterServlet}
import org.scalatra._
import javax.servlet.ServletContext


class ScalatraBootstrap extends LifeCycle {
  override def init(context: ServletContext) {
    context.mount(new GreeterServlet, "/*")
  }
}
```

`src/main/scala/com/example/app/GreeterServlet.scala`

```scala
package com.example.app

class GreeterServlet extends MyScalatraWebAppStack {
  get("/greet/:whom") {
    contentType = "text/html"
    val lucky =
      for (i <- (1 to 5).toList)
        yield util.Random.nextInt(48) + 1
    layoutTemplate("greeter_dry.scaml",
      "whom" -> params("whom"),
      "lucky" -> lucky)
  }
}
```

`src/main/webapp/WEB-INF/vies/greeter_dry.scaml`

```scala
-@ val whom: String
-@ val lucky: List[Int]
- attributes("title") = "Hello, "+whom

%h1 Congratulations
%p You've created your first Scalate view, #{whom}.
%p Your lucky numbers are:
%ul
  - for (number <- lucky)
    %li #{number}
```

`src/main/webapp/WEB-INF/layouts/default.scaml`

```scala
!!! 5
-@ val body: String
-@ val title: String = "foo"
%html
  %head
    %link(type="text/css" href="/css/style.css" rel="stylesheet")
    %title= title
  %body
    != body
```



效果如下



![](<https://counter2015.com/picture/scalatra-7-4.png>)

### [Twirl](https://github.com/spray/twirl)
另外一个模板引擎Twirl.



Play框架没有与scalate集成，而是创建了自己的模板系统。这与play框架的其余部分紧密结合在一起，但对其他Web框架很有吸引力。这促使spray.io团队将其拆分为一个名为twirl的单独项目。twirl模板有点像ssp，因为它们可以生成自由格式的文本，而不是严格的HTML结构。和scalate一样，twirl模板也被编译为在编译时尽可能多地捕获错误。



**配置**

添加sbt插件，需要在`project/plugins.sbt`中加入如下配置

```plain
addSbtPlugin("com.typesafe.sbt" % "sbt-twirl" % "1.3.13")
```

在build.sbt中启用插件

```plain
enablePlugins(SbtTwirl)
```



和Scalate不同，你会发现Twirl基本上就是纯的文本加上部分scala代码块

模板文件`src/main/twirl/greeting.scala.html`部分

```html
@(whom: String, lucky: List[Int])
<html>
    <head>
        <link type="text/css" href="/css/style.css" rel="stylesheet" />
        <title>Hello, @whom</title>
    </head>
    <body>
        <h1>Congratulations</h1>
        <p>You've created your first Twirl view, @whom.</p>
    <p>Your lucky numbers are:</p>
        <ul>
        @for(number <- lucky) {
            <li>@number</li>
        }
        </ul>
    </body>
</html>
```



调用部分

```scala
class GreeterServlet extends MyScalatraWebAppStack {
  get("/greet/:whom") {
    contentType = "text/html"
    val lucky =
      for (i <- (1 to 5).toList)
        yield util.Random.nextInt(48) + 1

    html.greeting(params("whom"), lucky)
  }
}
```

可以看出，这里有多了个约定，比如模板`a.scala.html`存放至`src/main/twirl`路径下，

调用时需要用`html.a`

使用这个模板最后网页的效果是一样的

> Twirl vs. Scalate
> The function call interface of Twirl offers a huge advantage over Scalate. Recall that
> Scalate passes attributes to its templates as a Map[String, Any] . Although the
> template itself is checked at compile time, the call to the template isn’t. In Twirl, it’s
> a compile-time error to call a template with the wrong parameters.
> Does this mean Twirl should replace Scalate? Not necessarily. Scalate is more widely
> used, and it supports many more dialects.
> Both template engines have their strengths. In a better world, we’d have Scalate’s
> wider dialect support merged with Twirl’s API



这是原书的话，大意是说两者各有优点，twirl可以更好地检查类型错误，而scalate支持的模板方言类型更多。

这两个模板用起来和markdown这种标记语言感觉差不多。。