# Scalatra in Action 1. hello, world

title: Scalatra in Action 1. hello, world
date: 2019-1-5 23:57:30
tags: [Scalatra, 读书笔记] 
categories: [技术]



------


## 前言

首先我们来介绍下我们的主角：[Scalatra][1]。

Scalatra是一个灵活而又可定制的轻量级框架，通过它你可以用Scala来编写后端服务。

你可能会问了，同样位置的框架有那么多，Python 有 Flask，tornado, Java有 Spring,<del>更别说世界上最好用的语言PHP了</del>就算同样是Scala开发，也有Play, Lift珠玉在前，Scalatra又有啥优势可言呢？

> Scalatra, with its minimalistic style, does one thing well: it makes HTTP actions trivially easy to express. Beyond that, it gets out of your way. This makes it easy to pick andchoose from tens of thousands of Java and Scala libraries to build exactly the application you need.  			——Scalatra in Action



> Scalatra is a simple, accessible and free web micro-framework.
>
> It combines the power of the JVM with the beauty and brevity of Scala, helping you quickly build high-performance web sites and APIs.										——scalatra.org

当你编程时，前端是一个绕不开的东西，当你做好了一个项目，像给别人展示，总不能拿着一长串莫名其妙的字符串，或者是一堆实点数说这就是你努力的成果，总归是要有一个界面的。

选择Scala是因为个人的喜好，喜欢那优雅简洁富有表达力的语句，而选择Scalatra则是看重它声称的**简单、轻量级**，<del>希望它不要像sbt一样</del>

废话少说，先用用看呗。

本文编写时，我对于Scalatra还一片茫然，后面学习地深入了，发现前面写错了，肯定会回来修改，但是也有可能有遗漏的东西，欢迎各位指正。

## 准备工作

工欲善其事，必先利其器。想要写程序，先得搭环境。

操作系统方面，选择lInux，具体的嘛，可以用ubuntu，也可以用Centos，可以用虚拟机，也可以用VPS，这里我为了方便直接操作，用的是Windows SubSystem Linux(简称WSL)。

要想建立Scalatra项目，首先需要一个JDK和一个sbt，sbt用来管理各种依赖，JDK用来将scala程序翻译成java字节码。

### Step 1: JDK

如果你之前安装过JDK，那么只要查看下jdk的版本就行了。

```shell
$ java -version
java version "1.8.0_181"
Java(TM) SE Runtime Environment (build 1.8.0_181-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.181-b13, mixed mode)
```



如果是JDK版本过低，还请升级到8，虽然说截至目前，JDK8已经停止了公共更新，最新的JDK已经更新到11了，但8目前还够用，等啥时候11的相关组件都配套了再升级过去，毕竟只是学习，对安全方面的要求没那么敏感。



如果你之前没有安装过JDK，可以从[Oracle 官网][2] 或者[Open JDK][3]下载需要的版本，具体安装过程网上到处都有，这里就不赘述了。



### Step 2: sbt

不同于大名鼎鼎的JDK，这里有必要简单介绍下[sbt][4]是个什么东西。(包含了我学习使用sbt过程中积累下来的深深的怨念，不想看的可以直接<a href="#1">跳过</a>)

sbt早先不叫sbt的，而叫SBT，除了大小写以外没有啥区别，看起来是吗？大写的时候一般来说代表缩写，当然，SBT在这里不是指sad but true，当项目初建时，是"Simple Build Tool",过了一段时间，SBT又有了"Scala Build Tools"的含义(sbt是Scala社区中*事实上的*构建工具，由[Lift web framework ](https://en.wikipedia.org/wiki/Lift_(web_framework))和[Play Framework使用](https://en.wikipedia.org/wiki/Play_Framework),当然它也能用来构建java项目)。

而现在sbt就叫*sbt*,还起了了个不明觉厉的绰号

> Nowadays we just call sbt “sbt”, and to reinforce that the name is no longer an [initialism](https://en.oxforddictionaries.com/definition/initialism) we always write it in all lowercase letters. However, we are cool with [酢豚](https://ja.wikipedia.org/wiki/%E9%85%A2%E8%B1%9A) (subuta) as a nickname.
>
> ​											——https://www.scala-sbt.org/1.x/docs/Faq.html



我用的时候倒是一点也没体会到"Simple Build Tool"和"Scala Build Tools"的含义。***Simple*** ?我从来没这么觉得，用得我整个人都觉得自己***Simple***了，在我看来，SBT是SB tools的缩写，先是展现其难用的特点，让你觉得自己是个SB，然后你用上一阵子，发现这东西真是个SB，包括但不限于以下几个问题:

- 下载包时经常会出现找不到包的情况，有两个可能，一个是由于众所周知的网络问题，解决办法科学上网；一个是由于默认的地址里面是maven中心仓库，要下载的包可能不在这里，这时候就要你手动添加下载地址了，那么下载地址咋填？这就看你个人的检索功底了<del>Bug驱动的面向搜索引擎编程</del>
- [warn] 	::::::::::::::::::::::::::::::::::::::::::::::
  [warn] 	::              FAILED DOWNLOADS            ::
  [warn] 	:: ^ see resolution messages for details  ^ ::
  [warn] 	::::::::::::::::::::::::::::::::::::::::::::::
  [warn] 	:: 
  [warn] 	::::::::::::::::::::::::::::::::::::::::::::::
- 上面这条Bug我遇到过不止一次，初见这个Bug的时候咋搜都搜不到，花了我一天时间，而现在都过去快一年了。截至当前(2019年),这个[bug](https://github.com/sbt/sbt/issues/3618) 还没有修复，只能靠社区提供的tricks暂时解决，而围绕它提出的issues有7个以上，。

至于其他的什么编译速度慢，打fat-jar时依赖冲突不好解决，就不提了，反正我每次写完`build.sbt`文件后都要把它当宝贝存着，指不定之后就要用上了。

“Scala Build Tools”也是，谁说只有你一家SBT了，Maven，gradle又不是吃干饭的。

怀着阴暗的想法揣度下，估计是官方也发现这个问题，才不好意思用SBT改成sbt（敢不敢叫Standard Buid Tool 呢）。

在我接触sbt前，从没想象过有这么一种构建工具，编译速度比eclipe慢，依赖问题比pip复杂，bug修复速度比IE还慢，sbt，你做到了(鼓掌)。

<del>要不是入了你的坑早用Gradle了</del>

或许等我进一步学习sbt后，或者版本更新后，会对其有所改观，但现在来说，sbt真TM难用。

 <a name="1">(跳过了一大段牢骚)</a>

sbt的安装过程可以参考[官方下载指导][5],选择对应的版本和操作系统平台，按照文档安装即可。

以CentOS为例

```shell
$ sudo yum install sbt
```



## 构建项目： Hello, World

### Giter8

> a command line tool to apply templates defined on GitHub



[Giter8][6]，简称g8，不像Kubernetes是因为中间有8个字母才简称k8s，g8很朴素，只取首尾两个字符作为简称。朴素的g8干的事情也很朴素，就是为程序生成模板。我们也可以用它来定制自定义的模板，这里不介绍如何自定义，只是简单使用。下面是一个简单的示例。

```shell
$ cd /root/test
$ sbt new eed3si9n/hello.g8
[info] Set current project to test (in build file:/root/test/)
[info] Set current project to test (in build file:/root/test/)
name [hello]: 
scala_version [2.11.8]: 

Template applied in /root/test/./hello

$ tree 
.
└── hello
    ├── README.markdown
    ├── app
    │   └── src
    │       └── main
    │           └── scala
    │               └── foo
    │                   └── Hello.scala
    ├── build.sbt
    └── project
        └── build.properties


```

要开始编写我们的第一个scalatra应用程序，我们需要借助g8来生成我们的模板。


{% spoiler eed3si9n是sbt的作者 %}


###  Hello, World

**生成scalatra项目**

```shel
$ sbt new scalatra/scalatra.g8
```

这会检查存放至github上的模板，并询问一些项目的配置项信息,`[]`内为默认值，比如`scalatra/scalatra.g8`的项目地址为https://github.com/scalatra/scalatra.g8

```shell
$ sbt new scalatra/scalatra.g8
[info] Set current project to scalatra-study (in build file:/mnt/d/IdeaProjects/scalatra-study/)
[info] Set current project to scalatra-study (in build file:/mnt/d/IdeaProjects/scalatra-study/)
organization [com.example]: com.github.counter2015
name [My Scalatra Web App]: My first Scalatra Web App
version [0.1.0-SNAPSHOT]: 
servlet_name [MyScalatraServlet]: 
package [com.example.app]: com.github.counter2015.app
scala_version [2.12.6]: 
sbt_version [1.2.1]: 
scalatra_version [2.6.4]: 

Template applied in /mnt/d/IdeaProjects/scalatra-study/./my-first-scalatra-web-app

$ tree 
.
└── my-first-scalatra-web-app
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
        │   │       └── github
        │   │           └── counter2015
        │   │               └── app
        │   │                   └── MyScalatraServlet.scala
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
                    └── github
                        └── counter2015
                            └── app
                                └── MyScalatraServletTests.scala

```

这里就不具体讲解项目目录和每个文件的含义了。

**编译**

```shell
$ cd my-first-scalatra-web-app
$ sbt
```

sbt首次编译时需要下载scalatra依赖的所有包，这需要花上一段时间。

```
sbt:My first Scalatra Web App> jetty:start
```

首次运行，下载[jetty][8]也需要花上一段时间。

{% spoiler jetty是一个Web服务器，如果你接触过Tomcat或者Nginx就很好理解 %}

```
2019-01-04 10:42:07.490:INFO:oejs.AbstractConnector:main: Started ServerConnector@865dd6{HTTP/1.1,[http/1.1]}{0.0.0.0:8080}
```

当出现以上日志时说明正常启动。

可以在http://localhost:8080/ 查看我们的第一个scalatra网页。

![](https://counter2015.com/picture/scalatra-1-1.jpg)

这个网页其实在`src/main/twirl/views/hello.scala.html`可以找到

```
@()
@layouts.html.default("Scalatra: a tiny, Sinatra-like web framework for Scala", "Welcome to Scalatra"){
  <p>Hello, Twirl!</p>
}
```

很奇怪的语法，对吧？那么谁调用的它呢？

```scala
// src/main/scala/com/github/counter2015/app/MyScalatraServlet.scala

package com.github.counter2015.app

import org.scalatra._

class MyScalatraServlet extends ScalatraServlet {

  get("/") {
    views.html.hello()
  }

}
```

那么最开始的调用者是谁呢？

```scala
// src/main/scala/ScalatraBootstrap.scala

import com.github.counter2015.app._
import org.scalatra._
import javax.servlet.ServletContext

class ScalatraBootstrap extends LifeCycle {
  override def init(context: ServletContext) {
    context.mount(new MyScalatraServlet, "/*")
  }
}
```



## 一些其他的尝试

如果我们不想在`8080`端口启动，该怎么做呢？	

build.sbt文件如下:

```scala
val ScalatraVersion = "2.6.4"

organization := "com.github.counter2015"

name := "My first Scalatra Web App"

version := "0.1.0-SNAPSHOT"

scalaVersion := "2.12.6"

resolvers += Classpaths.typesafeReleases

libraryDependencies ++= Seq(
  "org.scalatra" %% "scalatra" % ScalatraVersion,
  "org.scalatra" %% "scalatra-scalatest" % ScalatraVersion % "test",
  "ch.qos.logback" % "logback-classic" % "1.2.3" % "runtime",
  "org.eclipse.jetty" % "jetty-webapp" % "9.4.9.v20180320" % "container",
  "javax.servlet" % "javax.servlet-api" % "3.1.0" % "provided"
)

enablePlugins(SbtTwirl)
enablePlugins(ScalatraPlugin)
```

{% spoiler 在build.sbt中添加一行"containerPort in Jetty := 8090"，就能从8090启动 %}

> 每次代码更改后手动重新启动应用程序既缓慢又痛苦。使用自动代码重新加载工具可以轻松避免这种情况。
>
> sbt将允许您[在检测到代码更改时发出应用程序重新启动的信号](https://www.scala-sbt.org/1.x/docs/Triggered-Execution.html)。重新启动的语法涉及`~`在要重新执行的命令前添加。要自动重新编译和重新加载应用程序，请运行以下命令：
>
> ```shell
> $ sbt
> 
> > ~;jetty:stop;jetty:start
> ```

![](https://counter2015.com/picture/scalatra-1-2.jpg)





![](https://counter2015.com/picture/scalatra-1-3.jpg)

```shell
# 需要关闭时，在sbt中输入如下命令
jetty:stop
```



## 小结

我们初步了解了scalatra框架的用法，构建了第一个应用程序，但是也引入了新的问题

- scalatra没有main函数，启动过程到底是怎么做到的？
- 为什么`views.html.hello()`就能调用`src/main/twirl/views/hello.scala.html`的模板代码？
- `hello.scala.html`这种奇特的语法是什么？
- `twirl`是什么？

饭要一口一口吃，这些问题暂且放下，在继续学习的过程中，总会有解决的时刻。

至少我们能跑了一个Demo了，对吧（撒花<del>一篇hello world 水了这么多字</del>

## 参考资料

1. http://scalatra.org/getting-started/first-project.html
2. Scalatra in Action
3. 其实还有很多参考资料我没列出来是因为都在本文的链接里我就不想重新写了

[1]: https://en.wikipedia.org/wiki/Scalatra
[2]: http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
[3]: http://openjdk.java.net/install/index.html
[4]: https://en.wikipedia.org/wiki/Sbt_(software)#cite_note-8
[5]: https://www.scala-sbt.org/1.x/docs/Setup.html
[6]: http://www.foundweekends.org/giter8/index.html
[7]: https://github.com/eed3si9n/
[8]: https://www.eclipse.org/jetty/

