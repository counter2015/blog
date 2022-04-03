# Scalatra in Action 9. 配置、编译和部署

title: Scalatra in Action 9. 配置、编译和部署
date: 2020-04-19 19:20:30
tags: [Scalatra, 读书笔记] 
categories: [技术]



------




## 配置

为了让程序能适应各种不同的运行环境，我们需要通过配置文件，告诉它应该怎么做。

一般来说，应用程序会在以下三个环境下运行

- 开发环境
- 测试环境
- 生产环境

Scalatra内置有基于字符串的环境变量支持。我们可以用`ScalatraBase`中定义的`environment`方法来获取当前的运行环境，下面这段`environment`的代码是框架自带的

```scala
 def environment: String = {
    sys.props.get(EnvironmentKey) orElse initParameter(EnvironmentKey) getOrElse "DEVELOPMENT"
  }
```



如果没有设置这个值，那么默认的环境是`DEVELOPMENT`

```scala
get("/me") {
  environment match {
    case "DEVELOPMENT" => println("this is dev env")
    case "PRODUCTION" => println("this is pro env")
    case _ => println("unknown env")
  }
}
```

上面的代码可以帮助你在路由`/me`的为止查看当前使用的环境信息。

在这种情况下，Scalatra会显示更加详细的调试、错误信息。

如果想要修改这个环境，只要更改`org.scalatra.environmen`

有两个办法可以改

- 修改`src/main/webapp/WEB-INF/web.xml`
-  通过命令行传参给JVM传参`-Dorg.scalatra.environment=production`

> 冷知识： D在此处是define的意思

如果要通过修改`web.xml`来配置环境变量的话，可以参考下面的部分

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="http://java.sun.com/xml/ns/javaee"
    xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
    http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
    id="WebApp_ID" version="3.0">
                          
  <context-param>
    <param-name>org.scalatra.environment</param-name>
    <param-value>production</param-value>
  </context-param>
                            
  <listener>
    <listener-class>
    	org.scalatra.servlet.ScalatraListener
    </listener-class>
	</listener>
</web-app>
```

说实话，用xml的方式提供配置，眼睛负荷挺高的。

所以比较推荐用其他的配置库来管理项目配置。

这里推荐使用的是`lightbend`的配置

` https://github.com/lightbend/config `

可以到这个[网站](https://mvnrepository.com/ )查看对应的配置和版本号

在`build.sbt`中添加如下一行即可

`libraryDependencies += "com.typesafe" % "config" % "1.3.4"`

配置文件`application.conf`默认应该存放在`src/main/resources/application.conf`的位置

下面是一个示例

```conf
// port on which embedded servlet container should listen
port = 8080
// public url of the application
webBase = "http://dev.example.org"

// directory from which public web assets are served
assetsDirectory = "target/webapp"
// environment this instance runs in; can be one of development, test, staging, or production
environment = "development"

// sample email address
email {
 user = "my-app@example.com"
 password = "mypassword"
 host = "smtp.example.com"
 sender = "My App<my-app@example.com>"
}
```



具体的使用的话，可以先用`sbt console`尝试下

```scala
$ sbt console
scala> import com.typesafe.config.ConfigFactory
import com.typesafe.config.ConfigFactory

scala> val cfg = ConfigFactory.load
cfg: com.typesafe.config.Config = Config(...

scala> val webBase = cfg.getString("webBase")
webBase: String = http://dev.example.org

scala> println(webBase)
http://dev.example.org
```

之后的话，我们在项目中实例化配置类就能读取相应的配置项了。

下面是一个简单的构造示例

```scala
object AppConfig {
  def load: AppConfig = {
    val cfg = ConfigFactory.load
    val assetsDirectory = cfg.getString("assetsDirectory")
    val webBase = cfg.getString("webBase")
    val port = cfg.getInt("port")
    val env = AppEnvironment.fromString(cfg.getString("environment"))
    
    val mailConfig = MailConfig(
      cfg.getString("email.user"),
      cfg.getString("email.password"),
      cfg.getString("email.host"),
      cfg.getString("email.sender"))

    AppConfig(port, webBase, env, mailConfig, assetsDirectory)
  }
}
```



这里是剩下的`AppConfig`的对应代码

```scala
case class AppConfig(
                      port: Int,
                      webBase: String,
                      env: AppEnvironment,
                      mailConfig: MailConfig,
                      assetsDirectory: String
                    ) {

  def isProduction: Boolean = env == Production
  def isDevelopment: Boolean = env == Development
}

case class MailConfig(
  user: String,
  password: String,
  host: String,
  sender: String)

sealed trait AppEnvironment
case object Development extends AppEnvironment
case object Staging extends AppEnvironment
case object Test extends AppEnvironment
case object Production extends AppEnvironment
```

使用此配置也很简单，只需要在初始化的时候构造实例

```scala
class ScalatraBootstrap extends LifeCycle {
  override def init(context: ServletContext) {
    val conf = AppConfig.load
    sys.props(org.scalatra.EnvironmentKey) = AppEnvironment.asString(conf.env)
    context.mount(new MyScalatraServlet(conf), "/*")
  }
}
```

配置文件也可以用别的方法指定，比如指定文件系统的某个路径下的文件。

这部分可以参考对应的[文档](https://github.com/lightbend/config)。



## 编译

书上讲的编译过程已经有些过时了，这里我主要是结合原有的编译过程自己整理的步骤

还记得我们在最开始讲到的[sbt工具](http://counter2015.com/2019/01/05/scalatra1/#step-2-sbt)吗,下面的编译步骤中，主要需要用的就是这个工具

对于Scalatra项目的目录结构之前已经说明过了，忘记的可以翻看前面的文章

```scala
// ThisBuild define the scope, see https://www.scala-sbt.org/1.x/docs/Scopes.html
ThisBuild / organization := "com.example"
ThisBuild / name := "My Scalatra Web App"
ThisBuild / version := "0.1.0-SNAPSHOT"
ThisBuild / scalaVersion := "2.12.6"
ThisBuild / resolvers += Classpaths.typesafeReleases

val ScalatraVersion = "2.6.4"

lazy val defaultSettings = ScalatraPlugin.scalatraSettings ++ Seq(
  containerPort in Jetty := 8090,
  libraryDependencies ++= Seq(
    "org.scalatra" %% "scalatra" % ScalatraVersion,
    "org.scalatra" %% "scalatra-scalatest" % ScalatraVersion % "test",
    "ch.qos.logback" % "logback-classic" % "1.2.3" % "runtime",
    "org.eclipse.jetty" % "jetty-webapp" % "9.4.28.v20200408" % "container",
    "javax.servlet" % "javax.servlet-api" % "3.1.0" % "provided",
    "com.typesafe" % "config" % "1.3.4"
  )
)

lazy val web = (project in file("."))
  .enablePlugins(SbtTwirl, ScalatraPlugin)
  .settings(defaultSettings: _*)
```

上面是一个`build.sbt`的示例

这里加载了几个插件，在`project/plugins.sbt`中有配置

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-twirl" % "1.3.13")
addSbtPlugin("org.scalatra.sbt" % "sbt-scalatra" % "1.0.3")
```


## 部署

### 以 WAR方式部署

 WAR：Web application archive

参见：  https://en.wikipedia.org/wiki/WAR_(file_format) 

```shell
$ sbt package
```

打包后，在项目的targets目录下能找到`jar/war`的包

这里选择用jetty当作容器来部署，可以从[官网下载](https://www.eclipse.org/jetty/download.html)

安装步骤就不再赘述了

下载完成后，可以在`server.ini`（jetty版本9.4.28）修改对应的端口地址配置项`jetty.http.port`（默认是8080）

之前打包的war包，移动到webapps目录下，修改名称为`myweb.war`,此时可以在

 `http://localhost:8080/myweb/ `查看到服务运行状况



### 独立部署

独立部署时，需要修改`build.sbt`中`jetty`的作用范围，去除`container`

`"org.eclipse.jetty" % "jetty-webapp" % "9.4.28.v20200408"`

同时也需要对启动部分的代码做补充

```scala
import com.example.app._

import org.eclipse.jetty.server.{Server, ServerConnector}
import org.eclipse.jetty.webapp.WebAppContext
import org.scalatra.servlet.ScalatraListener

object ScalatraLauncher {
  def main(args: Array[String]): Unit = {
    val conf = AppConfig.load
    val server = new Server
    server.setStopTimeout(5000)
    server.setStopAtShutdown(true)


    val connector = new ServerConnector(server)
    connector.setHost("127.0.0.1")
    connector.setPort(conf.port)

    server.addConnector(connector)

    val webAppContext = new WebAppContext
    webAppContext.setContextPath("/")
    webAppContext.setResourceBase(conf.assetsDirectory)
    webAppContext.setEventListeners(Array(new ScalatraListener))

    server.setHandler(webAppContext)

    server.start()
    server.join()
  }
}
```

程序首先运行启动类`ScalatraLauncher`,加载配置文件，创建`Jetty`服务器并且设置绑定的host和端口。

接下来注册`WebAppContext`，这里会绑定`ScalatraListener`，它会监听`LifeCycle`的子类，加载里面对应的`MyScalatraServlet`



最后，`jetty`的服务启动并加入线程池

简单来说，打包只要两部操作，在`sbt shell`中依次执行`webappPrepare`和`dist`命令

前者会把需要的web应用内容移动到`target/webapp`文件夹

后者会把内容整理到`target/dist`文件夹，之后把独立部署所需要的所有内容进行打包。

```scala
sbt:web> help webappPrepare
prepare webapp contents for packaging
sbt:web> help dist
Build a distribution, assemble the files, create a launcher and make an archive.

sbt:web> webappPrepare
[success] Total time: 1 s, completed Apr 17, 2020 4:53:11 PM
sbt:web> dist
[info] Adding /mnt/d/IdeaProjects/scala/scalatra-study/my-scalatra-web-app/target/webapp to dist in /mnt/d/IdeaProjects/scala/scalatra-study/my-scalatra-web-app/target/dist/webapp
[success] Total time: 1 s, completed Apr 17, 2020 4:53:14 PM
```

此时可以观察下对应压缩包的内容（这里我起了个不太好的项目名`web`，可能会引起误解）

```shell
$ cp target/web-0.1.0-SNAPSHOT.zip ~/dev
$ cd ~/dev
$ unzip web-0.1.0-SNAPSHOT.zip 
$ cd web
$ tree
.
├── bin
│   └── web
├── lib
│   ├── ScalatraBootstrap.class
│   ├── ScalatraLauncher$.class
│   ├── ScalatraLauncher.class
│   ├── application.conf
│   ├── com
│   ├── commons-lang3-3.6.jar
│   ├── config-1.3.4.jar
│   ├── javax.servlet-api-3.1.0.jar
│   ├── jetty-http-9.4.28.v20200408.jar
│   ├── jetty-io-9.4.28.v20200408.jar
│   ├── jetty-security-9.4.28.v20200408.jar
│   ├── jetty-server-9.4.28.v20200408.jar
│   ├── jetty-servlet-9.4.28.v20200408.jar
│   ├── jetty-util-9.4.28.v20200408.jar
│   ├── jetty-webapp-9.4.28.v20200408.jar
│   ├── jetty-xml-9.4.28.v20200408.jar
│   ├── juniversalchardet-1.0.3.jar
│   ├── layouts
│   ├── logback-classic-1.2.3.jar
│   ├── logback-core-1.2.3.jar
│   ├── logback.xml
│   ├── mime-util-2.1.3.jar
│   ├── scala-library-2.12.6.jar
│   ├── scala-parser-combinators_2.12-1.0.6.jar
│   ├── scala-xml_2.12-1.0.6.jar
│   ├── scalatra-common_2.12-2.6.4.jar
│   ├── scalatra_2.12-2.6.4.jar
│   ├── slf4j-api-1.7.25.jar
│   ├── twirl-api_2.12-1.3.13.jar
│   └── views
└── webapp
    └── WEB-INF


```

| 名称        | 内容                 |
| ----------- | -------------------- |
| bin/web     | 服务启动脚本         |
| lib         | 主要存放各种jar包    |
| lib/layouts | 对应twirl中的layouts |
| lib/views   | 对应twirl中的views   |
| webapp      | 存放web.xml配置文件  |

生成的`bin/web`的内容如下

```shell
#!/bin/env bash
  
export CLASSPATH="lib:lib/scala-parser-combinators_2.12-1.0.6.jar:lib/scalatra_2.12-2.6.4.jar:lib/jetty-server-9.4.28.v20200408.jar:lib/jetty-servlet-9.4.28.v20200408.jar:lib/jetty-webapp-9.4.28.v20200408.jar:lib/config-1.3.4.jar:lib/jetty-io-9.4.28.v20200408.jar:lib/scala-xml_2.12-1.0.6.jar:lib/jetty-security-9.4.28.v20200408.jar:lib/javax.servlet-api-3.1.0.jar:lib/jetty-util-9.4.28.v20200408.jar:lib/commons-lang3-3.6.jar:lib/logback-classic-1.2.3.jar:lib/jetty-http-9.4.28.v20200408.jar:lib/juniversalchardet-1.0.3.jar:lib/slf4j-api-1.7.25.jar:lib/jetty-xml-9.4.28.v20200408.jar:lib/logback-core-1.2.3.jar:lib/scala-library-2.12.6.jar:lib/mime-util-2.1.3.jar:lib/twirl-api_2.12-1.3.13.jar:lib/scalatra-common_2.12-2.6.4.jar"
export JAVA_OPTS="-Xms1g -Xmx1g"


java $JAVA_OPTS -cp $CLASSPATH ScalatraLauncher
```

非常简单，把对应的jar包添加到运行的`CLASSPATH`里，然后启动主函数所在的类

当然，你也可以对配置稍作修改

```scala
val myDistSettings =
  DistPlugin.distSettings ++ Seq(
    exportJars := true,
    mainClass in dist := Some("ScalatraLauncher"),
    memSetting in dist := "2g",
    envExports in dist :=  Seq("LC_CTYPE=en_US.UTF-8", "LC_ALL=en_US.utf-8"),
    javaOptions in dist ++= Seq(
      "-Xss4m",
      "-Dfile.encoding=UTF-8",
      "-Dlogback.configurationFile=logback.xml",
      "-Dorg.scalatra.environment=production")
  )

lazy val web = (project in file("."))
  .enablePlugins(SbtTwirl, ScalatraPlugin)
  .settings(defaultSettings: _*)
  .settings(myDistSettings: _*)
```

启动很简单，把压缩包解压后，在项目路径下运行`bash bin/web`就可以了



### 用docker部署

在尝试若干次后，觉得用docker 打包java应用并不是很好

- 需要额外下载jdk
- 镜像文件臃肿
- 示例代码过于老旧
- wsl无法运行docker 重新安装了虚拟机

姑且是写了一个能打包的配置出来

```scala
// ThisBuild define the scope, see https://www.scala-sbt.org/1.x/docs/Scopes.html
ThisBuild / organization := "com.counter2015"
ThisBuild / name := "My Scalatra Web App"
ThisBuild / version := "0.1.0-SNAPSHOT"
ThisBuild / scalaVersion := "2.12.6"
ThisBuild / resolvers += Classpaths.typesafeReleases

val ScalatraVersion = "2.6.4"

lazy val defaultSettings = ScalatraPlugin.scalatraSettings ++ Seq(
  libraryDependencies ++= Seq(
    "org.scalatra" %% "scalatra" % ScalatraVersion,
    "org.scalatra" %% "scalatra-scalatest" % ScalatraVersion % "test",
    "ch.qos.logback" % "logback-classic" % "1.2.3" % "runtime",
    "org.eclipse.jetty" % "jetty-webapp" % "9.4.28.v20200408",
    "javax.servlet" % "javax.servlet-api" % "3.1.0" % "provided",
    "com.typesafe" % "config" % "1.3.4"
  )
)

val myDistSettings =
  DistPlugin.distSettings ++ Seq(
    exportJars := true,
    name := "web-example",
    mainClass := Some("ScalatraLauncher"),
    memSetting := "2g",
    envExports :=  Seq("LC_CTYPE=en_US.UTF-8", "LC_ALL=en_US.utf-8"),
    javaOptions ++= Seq(
      "-Xss4m",
      "-Dfile.encoding=UTF-8",
      "-Dlogback.configurationFile=logback.xml",
      "-Dorg.scalatra.environment=production")
  )

lazy val myDockerSettings = Seq(

  mainClass := Some("ScalatraLauncher"),

  exportJars := true,
  imageNames in docker := Seq(
    ImageName("sbtdocker/example:1.0.0"),
  ),

  dockerfile in docker := {

    val classpath = (fullClasspath in Runtime).value
    val webappDir = (target in webappPrepare).value

    val mainclass = mainClass.value.getOrElse(sys.error("Expected exactly one main class"))

    // Make a colon separated classpath with the JAR files
    val classpathString = classpath.files.map("/app/lib/" + _.getName).mkString(":")

    new Dockerfile {

      from("ubuntu:18.04")

      // Install packages
      
      // change apt source as aliyun
      runRaw("sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list")
      runRaw("apt-get clean")

      runRaw("apt-get update")
      runRaw("apt-get install -y vim curl wget unzip")

      // Install Oracle Java JDK 1.8.x
      runRaw("mkdir -p /usr/lib/jvm")
      runRaw("apt-get install -y default-jre")
      runRaw("apt-get install -y default-jdk")

      // Add all .jar files
      add(classpath.files, "/app/lib/")

      // All all files from webapp
      add(webappDir, "/app/webapp")

      // Remove lib (alternatively filter out before already)
      runRaw("rm -rf /app/webapp/WEB-INF/lib")

      // Define some volumes for persistent data (containers are immutable)
      volume("/app/conf")
      volume("/app/data")

      expose(80)

      workDir("/app")

      cmdRaw(
        f"java " +
          f"-Xmx4g " +
          f"-Dlogback.configurationFile=/app/conf/logback.xml " +
          f"-Dconfig.file=/app/conf/application.conf " +
          f"-cp $classpathString $mainclass")

    }
  }
)


lazy val webExample = (project in file("."))
  .enablePlugins(sbtdocker.DockerPlugin, SbtTwirl, ScalatraPlugin)
  .settings(defaultSettings: _*)
  .settings(myDistSettings: _*)
  .settings(myDockerSettings: _*)

```



打好包后，运行报错配置文件没有放对位置

运行脚本

```shell
#!/bin/bash
BASE=$(dirname $(readlink -f $0))
docker run -d \
-v $BASE/data:/app/data \
-v $BASE/conf:/app/conf:ro \
-p 8080:80 \
--name example1 sbtdocker/example:1.0.0
```

报错日志

```shell
$ docker logs example1 
Exception in thread "main" com.typesafe.config.ConfigException$IO: /app/conf/application.conf: java.io.FileNotFoundException: /app/conf/application.conf (No such file or directory)
	at com.typesafe.config.impl.Parseable.parseValue(Parseable.java:190)
	at com.typesafe.config.impl.Parseable.parseValue(Parseable.java:174)
	at com.typesafe.config.impl.Parseable.parse(Parseable.java:301)
	at com.typesafe.config.ConfigFactory.parseFile(ConfigFactory.java:739)
	at com.typesafe.config.DefaultConfigLoadingStrategy.parseApplicationConfig(DefaultConfigLoadingStrategy.java:51)
	at com.typesafe.config.ConfigFactory.defaultApplication(ConfigFactory.java:478)
	at com.typesafe.config.ConfigFactory$1.call(ConfigFactory.java:260)
	at com.typesafe.config.ConfigFactory$1.call(ConfigFactory.java:257)
	at com.typesafe.config.impl.ConfigImpl$LoaderCache.getOrElseUpdate(ConfigImpl.java:66)
	at com.typesafe.config.impl.ConfigImpl.computeCachedConfig(ConfigImpl.java:93)
	at com.typesafe.config.ConfigFactory.load(ConfigFactory.java:257)
	at com.typesafe.config.ConfigFactory.load(ConfigFactory.java:233)
	at com.example.app.AppConfig$.load(AppConfig.scala:52)
	at ScalatraLauncher$.main(ScalatraLauncher.scala:10)
	at ScalatraLauncher.main(ScalatraLauncher.scala)
Caused by: java.io.FileNotFoundException: /app/conf/application.conf (No such file or directory)
	at java.base/java.io.FileInputStream.open0(Native Method)
	at java.base/java.io.FileInputStream.open(FileInputStream.java:219)
	at java.base/java.io.FileInputStream.<init>(FileInputStream.java:157)
	at com.typesafe.config.impl.Parseable$ParseableFile.reader(Parseable.java:629)
	at com.typesafe.config.impl.Parseable.reader(Parseable.java:99)
	at com.typesafe.config.impl.Parseable.rawParseValue(Parseable.java:233)
	at com.typesafe.config.impl.Parseable.parseValue(Parseable.java:180)
	... 14 more

```



这部分等系统学习docker使用后再来尝试吧。

---
2022-04 更新
docker 打包可以使用 [sbt-native-packager](https://www.scala-sbt.org/sbt-native-packager/formats/docker.html)