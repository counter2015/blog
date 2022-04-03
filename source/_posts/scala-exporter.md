# 使用Scala开发应用程序 Prometheus Exporter

title: 使用Scala开发应用程序 Prometheus Exporter
date: 2019-05-17 21:59:30
tags: [Prometheus, scala] 
categories: [技术]

------

Prometheus官方提供了客户端API，包括

- Go
- Java or Scala
- Python
- Ruby

非官方的API包括但不限于

- Bash
- C++
- Erlang
- Haskell
- PHP
- ...

具体可以参照[官方文档](<https://prometheus.io/docs/instrumenting/clientlibs/>)这些语言中我最熟悉的是Scala，但是，github里和网上的例子大多都是Go的，

少有的部分Java例子也是用Spring Boot来做的<del>而我对这一窍不通</del>

经过漫长而细致的检索(Google, Github, 百度， Stackoverflow)，终于让我发现了一个能用的代码示例，虽然是Java的，

但是改改就能用了，[原文地址](<https://sysdig.com/blog/prometheus-metrics/>)中介绍有以下若干语言(Go, Java, Python, Javascript)的示例代码。

下面是一个简单的`build.sbt`文件写法示例

```scala
name := "PrometheusExporters"

version := "1.0.0"

scalaVersion := "2.11.12"

val prometheusVersion = "0.6.0"

libraryDependencies ++= Seq(
  "io.prometheus" % "simpleclient" % prometheusVersion,
  "io.prometheus" % "simpleclient_httpserver" % prometheusVersion,
  "io.prometheus" % "simpleclient_servlet" % prometheusVersion
)

assemblyMergeStrategy in assembly := {
  case PathList("META-INF", xs@_*) => MergeStrategy.discard
  case "application.conf" => MergeStrategy.concat
  case x =>
    val oldStrategy = (assemblyMergeStrategy in assembly).value
    oldStrategy(x)
}



```



具体代码如下

```scala
import java.io.IOException

import io.prometheus.client.exporter.HTTPServer
import io.prometheus.client.{Counter, Gauge, Histogram, Summary}

object Exporters {
  private def something(minValue: Int, maxValue: Int): Int = {
    val res = minValue + (Math.random() * (maxValue - minValue))
    res.toInt
  }

  def main(args: Array[String]): Unit = {
    val counter = Counter.build()
      .namespace("scala")
      .name("my_counter")
      .help("this is my counter")
      .register()

    val gauge = Gauge.build()
      .namespace("scala")
      .name("my_gauge")
      .help("this is my gauge")
      .register()

    val histogram = Histogram.build()
      .namespace("scala")
      .name("my_histogram")
      .help("this is my histogram")
      .register()

    val summary = Summary.build()
      .namespace("scala")
      .name("my_summary")
      .help("this is my summary")
      .register()

    import scala.language.implicitConversions
    implicit def somthingToRunnable(func: () => Unit): Runnable = new Runnable {
      def run(): Unit = func()
    }


    val bgThread = new Thread(() => {
      def randExporters(): Unit = {
        while (true) try {
          counter.inc(something(0, 5))
          println("counter value now is: " + counter.get())
          gauge.set(something(-5, 10))
          histogram.observe(something(0, 5))
          summary.observe(something(0, 5))
          Thread.sleep(1000)
        } catch {
          case e: InterruptedException =>
            e.printStackTrace()
        }
      }

      randExporters()
    })

    bgThread.start()
    try {
      val server = new HTTPServer(9090)
      println(s"server start at port ${server.getPort}")
    } catch {
      case e: IOException =>
        e.printStackTrace()
    }
  }
}
```

运行时需要依赖`prometheus`提供的包，所以最好直接使用`sbt assembly`打包。

