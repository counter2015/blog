# Scalatra in Action 10. 与数据库集成

title: Scalatra in Action 10. 与数据库集成
date: 2020-07-13 21:50:30
tags: [Scalatra, Slick, h2, 读书笔记] 
categories: [技术]



------

## 和数据库使用的场景

数据库是用来储存应用程序数据的通用方案，和数据库进行交互时，一般都是通过对应的库来做。



在Scalatra中并没有对某个具体的数据库做内建级别的支持，原则上来说，所有提供JVM连接库的数据库都能使用。

这一章主要介绍如何和Slick一起使用。



通过Slick,我们可以在Scala中不用使用裸的SQL语句，进行类型安全的SQL编程



Slick的官方文档可以在这里查看：https://scala-slick.org/docs/

入门教程：https://scala-slick.org/doc/3.3.2/gettingstarted.html

下面是一个例子，示例程序中描述的是登山，涉及到的表有两个，区域和路线

```scala
case class Area(
  id: Int,
  name: String,
  location: String,
  latitude: Double,
  longitude: Double,
  description: String)

case class Route(
  id: Int,
  areaId: Int,
  mountainName: Option[String],
  routeName: String,
  latitude: Double,
  longitude: Double,
  description: String)
```



## 添加依赖

首先需要添加`build.sbt`对应的依赖，这里增加的是`h2`和`slick`

```scala
libraryDependencies ++= Seq(
"com.typesafe.slick" %% "slick" % "3.3.2",
"com.h2database" % "h2" % "1.4.200") 
```

h2数据的官网：http://www.h2database.com/html/tutorial.html

由于Slick是全异步的，返回的结果是`Future[T]`, 不会阻塞当前操作。

需要添加一个`ExecutionContext`

```scala
import slick.driver.H2Driver.api._
import scala.concurrent.ExecutionContext.Implicits.global
```

为了和数据库进行交互，`Driver`帮我们把Slick的查询翻译成特定数据库的SQL

例如，`MySQL`的驱动为`MySQLDriver`,`H2`的驱动为`H2Driver`

多数驱动都是继承自`JdbcDriver`



## 使用SQL



`Database`代表了用来和数据库进行具体交互的对象，对应的操作类型是`DBIO[T]`

下面是一个例子，在内存中使用h2数据库

```scala
// multiple connections in one process
val jdbcUrl = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1"
val jdbcDriverClass = "org.h2.Driver"
val db = Database.forURL(jdbcUrl, driver = jdbcDriverClass)
```

`DB_CLOSE_DELAY=-1`参数表明，直到虚拟机关闭位置，数据库的内容都将保留。

每一个数据库的操作，都会被包装成`DBIO[T]`操作，`T`是操作的返回类型

下面用裸的SQL演示了一个创建数据表的例子

```scala
val createAction: DBIO[Int] = sqlu"create table foo(name varchar)"
val run: Future[Int] = db.run(createAction)
Await.result(run, Duration(5, "seconds"))
```



接下来是一个完整的例子，将多个操作合并执行

```scala
package com.example.app
import slick.dbio.DBIO

import scala.concurrent.{Await, Future}
import scala.concurrent.duration.Duration
import scala.concurrent.ExecutionContext.Implicits.global
import slick.jdbc.H2Profile.api._
object DB extends App {


  // Creates DBIO actions using sqlu interpolator
  val createFoo: DBIO[Int] = sqlu"create table foo(name varchar)"
  val dropFoo: DBIO[Int] = sqlu"drop table foo"

  // val selectFoo: DBIO[Seq[String]] = sql"select name from foo".as[String]
  def insertFoo(name: String): DBIO[Int] = sqlu"insert into foo values ($name)"

  // Queries data from the database
  val selectFoo: DBIO[Seq[String]] = sql"select name from foo".as[String]

  // Creates a composed action from other actions
  val composedAction: DBIO[Unit] = DBIO.seq(
    createFoo,
    insertFoo("air"),
    insertFoo("water"),
    selectFoo map { xs => xs foreach println },
    dropFoo
  )

  val jdbcUrl = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1"
  val jdbcDriverClass = "org.h2.Driver"
  val db = Database.forURL(jdbcUrl, driver = jdbcDriverClass)

  // Runs all statements of the composed action in a single transaction and waits for the result
  val run: Future[Unit] = db.run(composedAction)

  Await.result(run, Duration(5, "seconds"))
}
```

上面这段代码有一点需要注意的是，执行的时候带有输出的副作用	，下面是另外一个例子，在最后才输出结果

```scala
package com.example.app
import slick.dbio.DBIO

import scala.concurrent.{Await, Future}
import scala.concurrent.duration.Duration
import scala.concurrent.ExecutionContext.Implicits.global
import slick.jdbc.H2Profile.api._
object DB extends App {
  val jdbcUrl = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1"
  val jdbcDriverClass = "org.h2.Driver"
  val db = Database.forURL(jdbcUrl, driver = jdbcDriverClass)


  // Creates DBIO actions using sqlu interpolator
  val createFoo: DBIO[Int] = sqlu"create table foo(name varchar)"
  val dropFoo: DBIO[Int] = sqlu"drop table foo"

  // val selectFoo: DBIO[Seq[String]] = sql"select name from foo".as[String]
  def insertFoo(name: String): DBIO[Int] = sqlu"insert into foo values ($name)"

  // Queries data from the database
  val selectFoo: DBIO[Seq[String]] = sql"select name from foo".as[String]

  val composedAction: DBIO[Seq[String]] = for {
    _ <- createFoo
    _ <- insertFoo("air")
    _ <- insertFoo("water")
    xs <- selectFoo
    _ <- dropFoo
  } yield xs
  
  db.run(composedAction) foreach { xs => println(xs) } //Vector(air, water)
}
```

> Future[T]的结果需要使用Await才能获取

默认情况下，所以的语句都是以`auto-commit`的模式来运行的

这意味着一旦有一条语句出错，接下来的语句都不会被执行

而这会导致数据库处于一个不正确的状态下，为了避免这种情况，会把多个语句合并成事务

`db.run(composedAction.transactionally)`



上面演示了使用裸SQL的效果，尽管这样很灵活，但是缺不能保证类型安全

下面介绍的是用编程语言定义表的Schema

```scala
class Foos(tag: Tag) extends Table[(Int, String)](tag, "FOOS") {
  def id = column[Int]("INT", O.PrimaryKey, O.AutoInc)
  def name = column[String]("NAME")
  def * = (id, name)
}
```

这里定义了一个名为`Foos`的表，主键为`Int`类型，含有一个名为`NAME`的列

这里定义了函数方法`*`,是为了规定每一行的返回类型，比如这样返回的类型就是`Tuple2(Int, String)`

下面是最开始登山和路线两个抽象实体的具体表的定义

```scala
object Tables {
  class Areas(tag: Tag) extends Table[Area](tag, "AREAS") {
    def id = column[Int]("ID", O.PrimaryKey, O.AutoInc)
    def name = column[String]("NAME")
    def location = column[String]("LOCATION")
    def latitude = column[Double]("LATITUDE")
    def longitude = column[Double]("LONGITUDE")
    def description = column[String]("DESCRIPTION")
    def * = (id, name, location, longitude, latitude, description)
    <> (Area.tupled, Area.unapply)
  }
  class Routes(tag: Tag) extends Table[Route](tag, "ROUTES") {
    def id = column[Int]("ID", O.PrimaryKey, O.AutoInc)
    def areaId = column[Int]("AREAID")
    def mountainName = column[Option[String]]("MOUNTAINNAME")
    def routeName = column[String]("ROUTENAME")
    def description = column[String]("DESCRIPTION")
    def latitude = column[Double]("LATITUDE")
    def longitude = column[Double]("LONGITUDE")
    def * = columns <> (Route.tupled, Route.unapply)
    def area = foreignKey("FK_ROUTE_AREA", areaId, areas)(_.id)
  }
  val areas = TableQuery[Areas]
  val routes = TableQuery[Routes]
}
```

`<>`这个符号是干什么用的呢，简单来说就是为了方便从`case class`到`Tuple`直接的相互转换，

可以参考这个链接:https://stackoverflow.com/questions/35840626/what-does-the-operator-do-in-slick

```scala
case class User(id: Option[Int], first: String, last: String)
scala> User.tupled
res6: ((Option[Int], String, String)) => User = <function1>

scala> User.unapply _
res7: User => Option[(Option[Int], String, String)] = <function1>

val u = User(Some(1), "a", "b")
scala> User.unapply(u)
val res10: Option[(Option[Int], String, String)] = Some((Some(1),a,b))

User.tupled(res10.get)
val res13: User = User(Some(1),a,b)
```



`TableQuery[Areas]`有什么用呢？我们可以在这个基础上方便地进行数据库操作，比如建库删库<del>跑路</del>

```scala
import Tables.{routes, areas}
val createTables: DBIO[Unit] = (routes.schema ++ areas.schema).create
val dropTables: DBIO[Unit] = (routes.schema ++ areas.schema).drop
val insertArea: DBIO[Int] = {
  areas += Area(0, "Magic Wood", "Switzerland, Europe", 46.3347, 9.2612,
  "Hidden in the forest of the Avers valley there are some of the ...")
  
  // Same as INSERT INTO AREA(ID, NAME, LOCATION, LATITUDE, LONGTITUDE, DESCRIPTION)
  // VALUES (0, "Magic Wood", "Switzerland, Europe", 46.3347, 9.2612,
  "Hidden in the forest of the Avers valley there are some of the ...")
}
```

默认情况下， 插入操作会返回收到影响的结果条数

当某列被设置成自增属性时，插入时会被忽略

下面是一个多行插入的例子

```scala
def insertRoutes(areaId: Int): DBIO[Option[Int]] = {
  routes ++= Seq(
    Route(0, areaId, None, "Downunder", 46.3347, 9.2612, "Boulder, 7C+ at ..."),
    Route(0, areaId, None, "The Wizard", 46.3347, 9.2612, "Boulder, 7C at ..."),
    Route(0, areaId, None, "Master of a cow", 46.3347, 9.2612, "Boulder, 7C+ ...")
  )
}

val insertSampleData: DBIO[Int] = for {
  areaId <- insertAreaReturnId
  _ <- insertRoutes(areaId)
} yield areaId

val res: Future[Int] = db.run(insertSampleData)
res foreach println
```

执行`DBIO`操作时，返回的是`Future`，为了开启异步操作的支持，需要 mix in `FutureSupport` 这个`trait`

下面是和网页路由结合的一个示例

```scala
package com.example.app

import org.scalatra._
import slick.jdbc.H2Profile.backend.Database
import Tables.allAreas
import scala.concurrent.{ExecutionContext, Future}
import DbSetup.createArea
class MyScalatraServlet(db: Database) extends ScalatraServlet with FutureSupport {
  //Mixes in the FutureSupport trait and defines an implicit ExecutionContext for handling the Futures
  override protected implicit def executor: ExecutionContext = scala.concurrent.ExecutionContext.Implicits.global

  before("/*") {
    contentType = "text/html"
  }

  get("/areas") {

    new AsyncResult() {
      override val is: Future[_] = {
        db.run(allAreas map { areas =>
          views.html.areas(areas)
        })
      }
    }
  }

  get("/") {
    views.html.hello(UrlShortener.nextFreeToken)
  }

  post("/areas") {
    val name = params.get("name").getOrElse(halt(BadRequest()))
    val location = params.get("location") getOrElse halt(BadRequest())
    val latitude = params.getAs[Double]("latitude") getOrElse halt(BadRequest())
    val longitude = params.getAs[Double]("longitude") getOrElse halt(BadRequest())
    val description = params.get("description") getOrElse halt(BadRequest())

    db.run(createArea(name, location, latitude, longitude, description)) map { area =>
      Found(f"/areas/${area.id}")
    }
  }
}
```

渲染网页使用到的模板

```html
@import com.example.app._
@(areas: Seq[Area])
@areaUrl(uri: Int) = @{
    s"/areas/$uri"
}
@layouts.html.default("Scalatra: a tiny, Sinatra-like web framework for Scala", "Welcome to Scalatra") {
    <p>Hello, Twirl! </p>
    <div class = "row">
        <div class="col-md-8">
            @for(area <- areas) {
                <div class="panel panel-default">
                    <div class = "panel-heading">
                        Area: @area.name
                        <a pull-right href="@areaUrl(area.id)"> </a>
                    </div>
                </div>
                <div class = "panel-body">
                    @area.description
                </div>
            }
        </div>
    </div>
}
```

默认样式

```html
@(title: String, headline: String)(body: Html)
<html>
  <head>
    <title>@title</title>
    <link rel="stylesheet" type="text/css" href="/css/bootstrap.min.css" />
  </head>
  <body>
    <div class="container">
      <div class="row">
        <div class="col-md-8">
          <ul class="nav nav-pills">
            <li><a href="/">Home</a></li>
          </ul>
          <h2>@headline</h2>
        </div>
      </div>
      @body
    </div>
  </body>
</html>
```

这里使用的模板是`twirl`

官方文档：https://www.playframework.com/documentation/2.8.x/ScalaTemplates

运行效果如图

![](https://counter2015.com/picture/scalatra-10-1.jpg)

### 进一步使用SQL

我们在前面已经认识到了基础的查询，类型为`TableQuery[T]`

```scala
val areas = TableQuery[Areas]
val routes = TableQuery[Routes]
```

和Scala集合操作中的变换一样，我们对原始的query进行`map`或`filter`，会生成一个新的query，而不会对原来的query进行改写

比如

```scala
val routesQuery = routes.filter(_.areaId === 2).map(r => (r.id, r.routeName))
```

这段语句可以用for改写，也是等价的

```scala
val routesQuery = for {
  route <- routes
  if route.areaId == 2
} yield (route.id, route.routeName)
```



这个例子完整的为

```scala
package com.example.app

import slick.dbio.DBIO

import scala.concurrent.Await
import Tables._
import slick.jdbc.H2Profile.api._

import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration.Duration

object Demo {

  def main(args: Array[String]): Unit = {
    val db = Database.forConfig("h2mem1")

    val routes = TableQuery[Routes]
    val routesQuery =
      routes.filter(_.areaId === 2).map(r => (r.id, r.routeName))
    val routesAction: DBIO[Option[(Int, String)]] = routesQuery.result.headOption
    val action: DBIO[Option[(Int, String)]] = for {
      _ <- DbSetup.createDatabase
      x <- routesAction
      _ <- DbSetup.dropDatabase
    } yield x
    val res = Await.result(db.run(action), Duration(5, "seconds"))
    println(res)
    // Some((2,Bobcat-Houlihan))
  }
}
```

除此之外，你会感觉和普通的集合没什么区别

```scala
val lessRoutes = routes.drop(3).take(10)
val sortById = routes.sortBy(_.id.desc)
val sortByIdName = routes.sortBy(r => (r.areaId.asc, r.routeName))

val statistics = routes
  .groupBy(_.areaId)
  .map { case (areaId, rs) =>
    (areaId,
    rs.length,
    rs.map(_.longitude).min,
    rs.map(_.longitude).max,
    rs.map(_.latitude).min,
    rs.map(_.latitude).max)
  }
```

#### join操作

join操作可以划分成如下两种

- applicative join
- monadic join



**applicative join**

按照连接的键是否为空，可以再次划分为

- 内连接
- 外连接
- 左连接
- 右连接



```scala
val crossJoin = areas join routes
val innerJoin = routes join areas on (_.areaId === _.id)
val leftJoin = areas joinLeft routes on (_.id === _.areaId)
```



**monadic join**

上面的代码等效于

```scala
val innerJoinMonadic = for {
  r <- routes
  a <- areas if r.areaId === a.id
} yield (r, a)
```

在表格定义中，Routes中包含Area的外键

```scala
class Routes(tag: Tag) extends Table[Route](tag, "ROUTES") {
  ...
  def area = foreignKey("FK_ROUTE_AREA", areaId, areas)(_.id)
  ...
}
```

所以也可以通过外键来直接使用

```scala
val trailAreasQuery = for {
  route <- routes
  if (route.routeName.toLowerCase like "%trail")
  area <- route.area
} yield area
```



#### 数据的修改

下面介绍的是更新和删除操作

使用也很简单

```scala
val updateAction =
  routes.filter(_.routeName === "Midnight Lightning")
    .map(r => r.description)
    .update("Midnight Lightning is a problem on the Columbia Boulder.")

val deleteAction =
  routes.filter(_.routeName === "Midnight Lightning").delete
```



#### 常用查询的扩展集成

当SQL语句比较多的时候，可以考虑统一管理

这里是使用`implicit class`

```scala
  implicit class RoutesQueryExtensions[C[_]](query: Query[Routes, Route, C]) {

    val lessRoutes = query.drop(3).take(10)

    val sortById = query.sortBy(_.id.desc)
    val sortByIdName = query.sortBy(r => (r.areaId.asc, r.routeName))

    val statistics = query
      .groupBy(_.areaId)
      .map { case (areaId, rs) =>
        (areaId,
          rs.length,
          rs.map(_.longitude).min,
          rs.map(_.longitude).max,
          rs.map(_.latitude).min,
          rs.map(_.latitude).max)
      }

    def byId(id: Int) = query.filter(_.id === id)
    def byName(name: String) = query.filter(_.routeName === name)

    def bySuffix(str: String) =
      query.filter(_.routeName.toLowerCase like f"%%${str.toLowerCase}")

    val distinctAreaIds = query.groupBy(_.areaId).map(_._1)

    val withAreas = query join areas on (_.areaId === _.id)

  }
```

对应的演示部分

```scala
    def log(title: String)(xs: Seq[Any]): Unit = {
      println(f"$title")
      xs foreach { x => println(f"--$x") }
      println
    }


    val action = for {
      _ <- DbSetup.createDatabase
      _ <- routes.lessRoutes.result map log("limited routes")
      _ <- routes.sortById.result map log("sorted routes (id)")
      _ <- routes.sortByIdName.result map log("sorted routes (id, name)")
      _ <- routes.statistics.result map log("area statistics")
      _ <- DbSetup.dropDatabase
    } yield ()
```



完整的项目可以参照[此处](https://github.com/counter2015/example/tree/master/scalatra-slick-example)