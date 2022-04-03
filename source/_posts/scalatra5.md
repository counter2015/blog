# Scalatra in Action 5: 处理JSON

title: Scalatra in Action 5. 处理json
date: 2019-04-21 10:47:30
tags: [Scalatra, 读书笔记] 
categories: [技术]

------

长文预警

[JSON(JavaScript Object Notation)](<https://www.json.org/>)是一种运用非常广泛的数据格式，Scalatra也提供了对JSON的处理支持。

## 为应用程序添加JSON支持

在原项目`build.sbt`的合适位置添加如下依赖

```scala
libraryDependencies ++= Seq(
"org.scalatra" %% "scalatra-json" % ScalatraVersion,
"org.json4s" %% "json4s-jackson" % "3.3.0")
```

接下来我们来分析一段代码

```scala
import org.scalatra._
import org.scalatra.json._
import org.json4s._
import org.json4s.JsonDSL._

// mixes in the JacksonJsonSupport
trait MyJsonRoutes extends ScalatraBase with JacksonJsonSupport {
  // provide Json4s formats 
  implicit val jsonFormats: DefaultFormats.type = DefaultFormats
  get("/foods/foo_bar") {
    
    // produces a JSON JValue
    val productJson: JsonAST.JObject =
      ("label" -> "Foo bar") ~
        ("fairTrade" -> true) ~
        ("tags" -> List("bio", "chocolate"))
    productJson
  }
  post("/foods") {
      
    // Reads a tuple from the JSON request
    def parseProduct(jv: JValue): (String, Boolean, List[String]) = {
      val label = (jv \ "label").extract[String]
      val fairTrade = (jv \ "fairTrade").extract[Boolean]
      val tags = (jv \ "tags").extract[List[String]]
      (label, fairTrade, tags)
    }
      
    // invokes a simple parser
    val product = parseProduct(parsedBody)
    println(product)
  }
}
```

在上述代码中`JacksonJsonSupport`的`trait`负责将JSON请求与`JValue`进行互相转换。

```scala
// json4s 中的 相关源码  
case class JObject(obj: List[JField]) extends JValue {
    type Values = Map[String, Any]
    def values = obj.map { case (n, v) ⇒ (n, v.values) } toMap

    override def equals(that: Any): Boolean = that match {
      case o: JObject ⇒ obj.toSet == o.obj.toSet
      case _ ⇒ false
    }
      
 
```

在`get`部分的代码中返回的结果类型为`JValue`（此处存疑）,`JacksonJsonSupport`为代码提供`parsedBody`的方法，

它会将传入的JSON文本以JValue的形式返回，`parsedBody`做了以下几件事

- 从HTTP请求中提取JSON文本并转换成`JValue`
- 如果JSON格式有误返回`JNothing`
- 如果HTTP请求未将`Content-Type`头部设置为`application/json`

从请求中提取转换了JSON后，接下来要对里面的数据做如下处理

- 选择JSON文件中的某些部分
- 从数据结构中提取对应的值
- 处理数值丢失等问题

你也许注意到了这里有个`implict`的关键字，这是为了“告诉”json4s如何处理数据格式，举一个例子，JSON中的数字可以被视为`Double`或`BigDecimal`,这里我们使用的格式约定为`DefaultForamts`。

### JValue

JSON 文件的一个简单示例如下

```json
{
  "label" : "Foo bar",
  "fairTrade" : true,
  "tags" : [ "bio", "chocolate" ]
}
```

![](https://counter2015.com/picture/scalatra-5-1.jpg)

▲ 图片来源：scalatra in action

为了更方便的了解具体的过程，可以使用scala REPL来调试，但是这是运行时需要一些下载包，所以可以在`build.sbt`中已经配置好的项目依赖的情况下，到项目的根目录[通过sbt shell](<https://stackoverflow.com/questions/18812399/how-to-use-third-party-libraries-with-scala-repl>)来调试

```scala
$ sbt console
Welcome to Scala 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181).
Type in expressions for evaluation. Or try :help.

scala> import org.json4s._
import org.json4s._

scala> import org.json4s.JsonDSL._
import org.json4s.JsonDSL._

scala> val fooBar = JObject(
     | "label" -> JString("Foo bar"),
     | "fairTrade" -> JBool(true),
     | "tags" -> JArray(List(JString("bio"), JString("chocolate"))))
fooBar: org.json4s.JsonAST.JObject = JObject(List((label,JString(Foo bar)), (fairTrade,JBool(true)), (tags,JArray(List(JString(bio), JString(chocolate)))))

scala> import org.json4s.jackson.JsonMethods.parse
import org.json4s.jackson.JsonMethods.parse

scala> val txt = 
     |   """{
     | | "tags": ["bio","chocolate"],
     | | "label": "Foo bar",
     | | "fairTrade": true
     | |}""".stripMargin
txt: String =
{
 "tags": ["bio","chocolate"],
 "label": "Foo bar",
 "fairTrade": true
}

scala> val parsed = parse(txt)
parsed: org.json4s.JValue = JObject(List((tags,JArray(List(JString(bio), JString(chocolate)))), (label,JString(Foo bar)), (fairTrade,JBool(true))))

scala> fooBar == parsed
res0: Boolean = true
```
从这段代码没可以看出，先前定义的结构体`fooBar`和由字符串文本转换而来的`parsed`相等，并且注意到在`JObject`中的值的顺序对相等的判断不会造成影响。



## JSON的输入输出

当你的程序接收到JSON请求时，需要对其进行加工处理

- 从HTTP请求中提取JSON信息
- 确定某个字段的JSON值
- 处理提取错误异常

下面是一段JSON文本

```json
{
  "title": "Penne with cocktail tomatoes, Rucola and Goat cheese",
  "details": {
    "cuisine": "italian",
    "vegetarian": true
  },
  "ingredients": [{
    "label": "Penne",
    "quantity": "250g"
  }, {
    "label": "Cocktail tomatoes",
    "quantity": "300g"
  }, {
    "label": "Rucola",
    "quantity": "2 handful"
  }, {
    "label": "Goat cheese",
    "quantity": "200g"
  }, {
    "label": "Garlic cloves",
    "quantity": "2 tsps"
  }],
  "steps": [
    "Cook noodles until aldente.",
    "Quarter the tomatoes, wash the rucola, dice
      the goat's cheese and cut the garlic.",
    "Heat olive oil in a pan, add the garlic and the tomatoes and
      steam short (approx. for 5 minutes).",
    "Shortly before the noodles are ready add the rucola
      to the tomatoes.",
    "Drain the noodles and mix with the tomatoes,
      finally add the goat's cheese and serve."
  ]
}
```

尽管可以使用`JValue`来处理JSON数据，但是如果把JSON包装到类中，不仅能提供类型安全方面的便利和提高可读性，而且可以方便数据库的直接映射。因而，构造如下三个类

```scala
case class Recipe(title: String,
                  details: RecipeDetails,
                  ingredients: List[IngredientLine],
                  steps: List[String])
case class RecipeDetails(cuisine: String, vegetarian: Boolean,
                         diet: Option[String])
case class IngredientLine(label: String, quantity: String)
```



通过`Json4s`这个库来构造`JValue`有三种方法

- 使用`JValue`类型
- 使用`JValue`的[DSL](<https://en.wikipedia.org/wiki/Domain-specific_language>)
- 使用通用的基于反射的方法将值映射成为`Jvalue`

之前上面那一大串代码就是基于第一种方法来构造`JValue`的，这种办法的优点在于显式地表现出了过程，但也因为这样让程序变得十分啰嗦。

DSL的方法需要用到一些操作符和隐式转换，为了启用基本类型的转换推断，需要导入`org.json4s.JsonDSL._ `

```scala
scala> import org.json4s.JsonDSL._
import org.json4s.JsonDSL._

scala> import org.json4s.JValue
import org.json4s.JValue

scala> val jsString: JValue = "something"
jsString: org.json4s.JValue = JString(something)

scala> val jsBool: JValue = true
jsBool: org.json4s.JValue = JBool(true)

scala> val detailsJson = ("cuisine" -> "italian") ~ ("vegetarian" -> true)
detailsJson: org.json4s.JsonAST.JObject = JObject(List((cuisine,JString(italian)), (vegetarian,JBool(true))))

scala> val tags: JValue = List("higher", "cuisine")
tags: org.json4s.JValue = JArray(List(JString(higher), JString(cuisine)))


```

JSON Object可以从`Tuple2[String, A]`构造来，A在这里默认被视为`JValue`。`~`运算符将多个语句连接成一个JSON Object。

还记得之前那段很长的JSON文本吗，下面用DSL来构造`JValue`(接着上文的环境继续)

```scala
scala> val recipeJson = (
     |   "title" -> "Penne with cocktail tomatoes, Rucola and Goat cheese") ~ (
     |   "details" -> detailsJson) ~ (
     |   "ingredients" -> List(
     |     ("label" -> "Penne") ~ ("quantity" -> "250g"),
     |     ("label" -> "Cocktail tomatoes") ~ ("quantity" -> "300g"),
     |     ("label" -> "Rucola") ~ ("quantity" -> "2 handful"),
     |     ("label" -> "Goat cheese") ~ ("quantity" -> "250g"),
     |     ("label" -> "Garlic cloves") ~ ( "quantity" -> "250g"))) ~ (
     |   "steps" -> List(
     |     "Cook noodles until aldente.",
     |     "Quarter the tomatoes, wash the rucola,dice the goat's cheese ...",
     |     "Heat olive oil in a pan, add the garlic and the tomatoes and ...",
     |     "Shortly before the noodles are ready add the rucola to the ...",
     |     "Drain the noodles and mix with the tomatoes,finally add the ..."))
recipeJson: org.json4s.JsonAST.JObject = JObject(List((title,JString(Penne with cocktail tomatoes, Rucola and Goat cheese)), (details,JObject(List((cuisine,JString(italian)), (vegetarian,JBool(true))))), (ingredients,JArray(List(JObject(List((label,JString(Penne)), (quantity,JString(250g)))), JObject(List((label,JString(Cocktail tomatoes)), (quantity,JString(300g)))), JObject(List((label,JString(Rucola)), (quantity,JString(2 handful)))), JObject(List((label,JString(Goat cheese)), (quantity,JString(250g)))), JObject(List((label,JString(Garlic cloves)), (quantity,JString(250g))))))), (steps,JArray(List(JString(Cook noodles until aldente.), JString(Quarter the tomatoes, wash the rucola,dice the goat's cheese ...), JString(Heat olive oil in a pan, add the garlic and the tomatoes and ...), J...

```

使用DSL了好处之一是可以用简短的代码来构造`JValue`，但是这里看起来还是太啰嗦了，用我们之前定义的类来构造如何？

```scala
scala> case class RecipeDetails(cuisine: String, vegetarian: Boolean, diet: Option[String])
defined class RecipeDetails

scala> val jsObject: JValue =  ("details" -> RecipeDetails("italian", true, None))
<console>:20: error: type mismatch;
 found   : (String, RecipeDetails)
 required: org.json4s.JValue
    (which expands to)  org.json4s.JsonAST.JValue
       val jsObject: JValue =  ("details" -> RecipeDetails("italian", true, None))

```

这里报错了，因为我们没有给自定义的`RecipeDetails`类声明其隐式类型转换。

```scala
scala> implicit val formats = DefaultFormats
formats: org.json4s.DefaultFormats.type = org.json4s.DefaultFormats$@4fe1e61b

scala> import scala.language.implicitConversions
import scala.language.implicitConversions

scala> implicit def details2JValue(rd: RecipeDetails): JValue = Extraction.decompose(rd)
warning: there was one feature warning; re-run with -feature for details
details2JValue: (rd: RecipeDetails)org.json4s.JValue

scala> val jsObject: JValue =  ("details" -> RecipeDetails("italian", true, None))
jsObject: org.json4s.JValue = JObject(List((details,JObject(List((cuisine,JString(italian)), (vegetarian,JBool(true)))))))
```



这里函数`details2JValue`依赖于`decompose`将`RecipeDetails`转换成`JValue`的类型，`decompose`是一种常用的基于反射的方法，它可以通过

函数`Extraction.decompose(x: Any): JValue`将各种类转换成`JVAlue`

转换过程中遵循以下规则

- 基本类型会转换成其对应的JSON基本类型，如String会转换成JString
- 集合类型转换成JArray
- Object转换成JObject

但是也可以通过自己设置的转换器改写默认转换规则。



下面来说明如何处理JSON。

（你最好还没有关闭我们之前的sbt console

```scala
scala> val detailsJson = ("cuisine" -> "italian") ~ ("vegetarian" -> true)

scala> val recipeJson = (
     |         "title" -> "Penne with cocktail tomatoes, Rucola and Goat cheese") ~ (
     |         "details" -> detailsJson) ~ (
     |         "ingredients" -> List(
     |           ("label" -> "Penne") ~ ("quantity" -> "250g"),
     |           ("label" -> "Cocktail tomatoes") ~ ("quantity" -> "300g"),
     |           ("label" -> "Rucola") ~ ("quantity" -> "2 handful"),
     |           ("label" -> "Goat cheese") ~ ("quantity" -> "250g"),
     |           ("label" -> "Garlic cloves") ~ ( "quantity" -> "250g"))) ~ (
     |         "steps" -> List(
     |           "Cook noodles until aldente.",
     |           "Quarter the tomatoes, wash the rucola,dice the goat's cheese ...",
     |           "Heat olive oil in a pan, add the garlic and the tomatoes and ...",
     |           "Shortly before the noodles are ready add the rucola to the ...",
     |           "Drain the noodles and mix with the tomatoes,finally add the ..."))



scala> val res1 = recipeJson \ "title"
res1: org.json4s.JValue = JString(Penne with cocktail tomatoes, Rucola and Goat cheese)

scala> val res2 = recipeJson \ "details" \ "cuisine"
res2: org.json4s.JValue = JString(italian)

scala> val res3 = recipeJson \ "cuisine" \ "details"
res3: org.json4s.JValue = JNothing

scala> val res4 = recipeJson \ "details" 
res4: org.json4s.JValue = JObject(List((cuisine,JString(italian)), (vegetarian,JBool(true))))




```

这个函数`\(nameToFind: String): JValue`会按指定的`nameToFind`从外到里依次查询，它能返回一个Object，也能返回一个数组，还能做合并操作。当查询的值不存在时，返回的是`JNothing`

我们也可以简单地在所有的区域查找对应的key，对应的函数为`\\(nameToFind: String)`

```scala
scala> val data1 = ("a" -> "1") ~ ("b" -> List("a" -> false))

scala> data1 \\ "a"
res0: org.json4s.JValue = JObject(List((a,JString(1)), (a,JBool(false))))

scala> data1 \\ "c"  
res6: org.json4s.JValue = JObject(List())

scala> data1 \\ "c" \ "d" 
res5: org.json4s.JValue = JNothing

```

是不是感觉和`flatMap`有点像？

这里值得注意的是JNothing不是Null，表示空值要用到JNull(这里等以后弄明白的可以详细说下)

这里提取的是json4s的类型，要转换成能计算的类型，可以用`extract[A]`

可以提取到以下几个目标

- case Classes or Classes
- 基本类型
- 标准的集合类型
- 任何有实现自定义反序列化的类型

当从`JValue`中提取数值时，会检查`JValue`和目标类型是否合适，如果确认格式匹配，那么会按照目标的类型来构造

```scala
scala> (recipeJson \ "title").extract[String]
res11: String = Penne with cocktail tomatoes, Rucola and Goat cheese

scala> (recipeJson \ "steps").extract[List[String]]
res13: List[String] = List(Cook noodles until aldente., Quarter the tomatoes, wash the rucola,dice the goat's cheese ..., Heat olive oil in a pan, add the garlic and the tomatoes and ..., Shortly before the noodles are ready add the rucola to the ..., Drain the noodles and mix with the tomatoes,finally add the ...)

scala> (recipeJson \ "details").extract[RecipeDetails]
res14: RecipeDetails = RecipeDetails(italian,true,None)

```



如果强行提取不合适的类型，程序会抛出异常

```scala
scala> JNothing.extract[String]
org.json4s.package$MappingException: Did not find value which can be converted into java.lang.String
  at org.json4s.reflect.package$.fail(package.scala:95)
  at org.json4s.Extraction$$anonfun$org$json4s$Extraction$$convert$2.apply(Extraction.scala:744)
  at org.json4s.Extraction$$anonfun$org$json4s$Extraction$$convert$2.apply(Extraction.scala:744)
  at scala.Option.getOrElse(Option.scala:121)
  at org.json4s.Extraction$.org$json4s$Extraction$$convert(Extraction.scala:744)
  at org.json4s.Extraction$$anonfun$extract$7.apply(Extraction.scala:403)
  at org.json4s.Extraction$$anonfun$extract$7.apply(Extraction.scala:401)
  at org.json4s.Extraction$$anonfun$customOrElse$1.apply(Extraction.scala:646)
  at org.json4s.Extraction$$anonfun$customOrElse$1.apply(Extraction.scala:646)
  at scala.PartialFunction$class.applyOrElse(PartialFunction.scala:123)
  at scala.collection.AbstractMap.applyOrElse(Map.scala:59)
  at org.json4s.Extraction$.customOrElse(Extraction.scala:646)
  at org.json4s.Extraction$.extract(Extraction.scala:401)
  at org.json4s.Extraction$.extract(Extraction.scala:40)
  at org.json4s.ExtractableJsonAstNode.extract(ExtractableJsonAstNode.scala:21)
  ... 40 elided

```

为了以放这种情况的发生，数据可以用`Option`的方式包装起来,

使用的方法为`extractOpt[A](json: JValue): Option[A]`,可能会返回以下结果

- Some(v) 如果能被正常提取
- None 
  - 如果提取失败
  - 如果调用者为`JNull`或者`JNothing`

```scala
scala> JString("foo").extractOpt[String]
res16: Option[String] = Some(foo)

scala> JString("foo").extractOpt[Boolean]
res17: Option[Boolean] = None

scala> JNull.extractOpt[String]
res18: Option[String] = None

scala> JNothing.extractOpt[String]
res19: Option[String] = None
```

<del>下面进入到天书环节</del>

一个可选的提取方法是使用[scalaz](<https://github.com/scalaz/scalaz>)的`\/`类型，当提取成功时返回`Ok(...)`,否则返回`BadRequest()`

```scala
import org.scalatra._
import org.scalatra.json._
import org.json4s._
import scalaz.{-\/, \/-}
import  scalaz.Scalaz._
trait MyJsonScalazRoutes extends ScalatraBase with JacksonJsonSupport {
  // be able to handle scalaz' \/ as return value, simply unwrap the value from the container
  override def renderPipeline: RenderPipeline = ({
    case \/-(r) => r
    case -\/(l) => l
  }: RenderPipeline) orElse super.renderPipeline

  post("/foods_alt") {
    for {
      // short-circuits with BadRequest when optional extraction fails
      label <- (parsedBody \ "label").extractOpt[String] \/> BadRequest()
      fairTrade <- (parsedBody \ "fairTrade").extractOpt[Boolean] \/> BadRequest()
      tags <- (parsedBody \ "tags").extractOpt[List[String]]  \/> BadRequest()
    } yield Ok((label, fairTrade, tags))
    // yields an ok when all previous steps succeed
  }
}
```

是不是觉得符号鬼畜看不懂？`\/-`是啥，`-\/`又是啥,`\/>`又是啥，为了更好地理解，来看下它们的定义

```scala
// scalaz.Either.scala
package scalaz

import scala.util.control.NonFatal
import scala.reflect.ClassTag
import Liskov.<~<

/** Represents a disjunction: a result that is either an `A` or a `B`.
 *
 * An instance of `A` [[\/]] B is either a [[-\/]]`[A]` (aka a "left") or a [[\/-]]`[B]` (aka a "right").
 *
 * A common use of a disjunction is to explicitly represent the possibility of failure in a result as opposed to
 * throwing an exception. By convention, the left is used for errors and the right is reserved for successes.
 * For example, a function that attempts to parse an integer from a string may have a return type of
 * `NumberFormatException` [[\/]] `Int`. However, since there is no need to actually throw an exception, the type (`A`)
 * chosen for the "left" could be any type representing an error and has no need to actually extend `Exception`.
 *
 * `A` [[\/]] `B` is isomorphic to `scala.Either[A, B]`, but [[\/]] is right-biased for all Scala versions, so methods
 * such as `map` and `flatMap` apply only in the context of the "right" case. This right bias makes [[\/]] more
 * convenient to use than `scala.Either` in a monadic context in Scala versions <2.12. Methods such as `swap`,
 * `swapped`, and `leftMap` provide functionality that `scala.Either` exposes through left projections.
 *
 * `A` [[\/]] `B` is also isomorphic to [[Validation]]`[A, B]`. The subtle but important difference is that [[Applicative]]
 * instances for [[Validation]] accumulates errors ("lefts") while [[Applicative]] instances for [[\/]] fail fast on the
 * first "left" they evaluate. This fail-fast behavior allows [[\/]] to have lawful [[Monad]] instances that are consistent
 * with their [[Applicative]] instances, while [[Validation]] cannot.
 */



/** A left disjunction
 *
 * Often used to represent the failure case of a result
 */
final case class -\/[+A](a: A) extends (A \/ Nothing)

/** A right disjunction
 *
 * Often used to represent the success case of a result
 */
final case class \/-[+B](b: B) extends (Nothing \/ B)


//scalaz.syntax.std.OptionOps.scala
final def \/>[E](e: => E): E \/ A = o.toRight(self)(e)

final def <\/[B](b: => B): A \/ B = o.toLeft(self)(b)

//scalaz.std.Option.scala
final def toRight[A, E](oa: Option[A])(e: => E): E \/ A = oa match {
  case Some(a) => \/-(a)
  case None    => -\/(e)
}

final def toLeft[A, B](oa: Option[A])(b: => B): A \/ B = oa match {
  case Some(a) => -\/(a)
  case None    => \/-(b)
}

//org.scalatra.ActionResult.scala
object BadRequest {
  def apply(body: Any = Unit, headers: Map[String, String] = Map.empty) =
    ActionResult(400, body, headers)
}

object Ok {
  def apply(body: Any = Unit, headers: Map[String, String] = Map.empty) =
    ActionResult(200, body, headers)
}
```

Begin of 个人理(cai)解(xiang)（不保证正确性

A \/ B 和 scala.Either[A,B]同构，返回的结果是[[-\/]]`[A]` ，或者是 [[\/-]]`[B]` .

前者一般用于返回错误的情况，后者一般用于返回正确的情况

上述代码首先定义了一个`RenderPipeline`用scalaz的符号重写了下匹配成功、失败情况下的映射关系，

同时，在scalaz符号定义范围外的部分使用默认的规则来做映射。

在`post`代码内部，使用`for`循环提取标签并返回结果，如果没有问题，返回的结果为HTTP  200，失败情况下，`extractOpt`

提取的结果为None,此时通过Scalaz显式地把异常转换成`BadRequest`,返回地结果为HTTP 400

End of 个人理解



上面的代码给出了大部分情况下对JSON提取用的方法，下面来讨论一些特定的情况。

## 处理匹配失败，定制JSON支持

在类和JSON对象的结构非常相似的情况下，上一节介绍的转换就足够了。事实上，经常会有某种形式的差异。这可以是字段命名的简单变化，也可以是结构的差异。

有时你可以调整一侧使两者匹配，但在某些情况下这可能是不可行的。例如，当使用现有的数据模型时，由于依赖关系，无法进行代码重构。同样，当支持标准化的JSON格式时，设计scala类以完全匹配JSON格式可能是不切实际的。

在这些情况下，自定义转换函数是一个更具有性价比的选择。



### 定制JSON格式

正如之前看到的，要想直接提取JSON中的值，需要设置`Formats`的值

`DefaultFormats`是一个实现了默认公共值的`trait`,下面是它会生效的场合

- 以非默认的格式读写日期
- 从JSON中读取`BigDecimal`而不是`Double`
- 当从JSON中提取某个字段类型时需要用自定义序列化反序列化做转化时

下面是一个在默认格式下处理时间的例子

```scala
scala> import org.json4s._
import org.json4s._

scala> import org.json4s.Extraction.decompose
import org.json4s.Extraction.decompose

scala> import org.json4s.jackson.JsonMethods.{parse, compact}
import org.json4s.jackson.JsonMethods.{parse, compact}

scala> import java.util.Date
import java.util.Date

scala> implicit val formats = DefaultFormats
formats: org.json4s.DefaultFormats.type = org.json4s.DefaultFormats$@48536f7d

scala> val txt = compact(decompose(Map("date" -> new Date())))
txt: String = {"date":"2019-04-30T03:37:36Z"}

scala> val date = (parse(txt) \ "date").extractOpt[Date]
date: Option[java.util.Date] = Some(Tue Apr 30 03:37:36 GMT 2019)

```

当然我们可以自定义解析的格式

```scala
scala> import java.text.SimpleDateFormat
import java.text.SimpleDateFormat

scala> implicit val formats = new DefaultFormats {
     |   override def dateFormatter: SimpleDateFormat = {
     |     new SimpleDateFormat("yyyy-MM-dd")
     |   }
     | }
formats: org.json4s.DefaultFormats = $anon$1@23a4dc4f

scala> val txt = compact(decompose(Map("date" -> new Date())))
txt: String = {"date":"2019-04-30"}

scala> val date = (parse(txt) \ "date").extractOpt[Date]
date: Option[java.util.Date] = Some(Tue Apr 30 00:00:00 GMT 2019)

```

![](https://counter2015.com/picture/scalatra-5-2.png)

```scala
scala> import org.json4s._
import org.json4s._

scala> import org.json4s.JsonDSL._
import org.json4s.JsonDSL._

scala> import org.json4s.jackson.Serialization.write
import org.json4s.jackson.Serialization.write

scala> implicit val formats = DefaultFormats
formats: org.json4s.DefaultFormats.type = org.json4s.DefaultFormats$@48536f7d

scala> class Foo(x: String, val y: String) {
     |   private val a: String = "a"
     |   var b: String = "b"
     | }
defined class Foo

scala> val foo = new Foo("x", "y")
foo: Foo = Foo@6cb7cba

scala> val txt1 = Serialization.write(foo)
txt1: String = {"y":"y"}

```

从上面的例子中，可以发现只有`y`被成功提取，而`Foo`函数有三个成员变量`a,b,y`，通过`FieldSerializer[Foo]`可以取出所有字段

```scala
scala> import org.json4s._
import org.json4s._

scala> import org.json4s.JsonDSL._
import org.json4s.JsonDSL._

scala> import org.json4s.jackson.Serialization
import org.json4s.jackson.Serialization

scala> implicit val formats = DefaultFormats + new FieldSerializer[Foo]()
formats: org.json4s.Formats = org.json4s.Formats$$anon$3@3494281a

scala> val foo = new Foo("x", "y")
foo: Foo = Foo@49f9288

scala> val txt1 = Serialization.write(foo)
txt1: String = {"y":"y","a":"a","b":"b"}

```

### 处理多态

下面是使用类型提示来处理多态的例子

假设你有如下几个类

```scala
sealed trait Measure
case class Gram(value: Double) extends Measure
case class Teaspoon(value: Double) extends Measure
case class Tablespoon(value: Double) extends Measure
case class Handful(value: Double) extends Measure
case class Pieces(value: Double) extends Measure
case class Milliliter(value: Double) extends Measure
```



> A sealed class cannot have any new subclasses added except the ones in the same file



```scala
import org.json4s._
import org.json4s.JsonDSL._
import org.json4s.Extraction.decompose

implicit val formats = DefaultFormats

val amounts = List(Handful(2), Gram(300), Teaspoon(2))
// amounts: List[Product with Serializable with Measure] = List(Handful(2.0), Gram(300.0), Teaspoon(2.0))

val amountsJson = Extraction.decompose(amounts)
// amountsJson: org.json4s.JValue = JArray(List(JObject(List((value,JDouble(2.0)))), JObject(List((value,JDouble(300.0)))), JObject(List((value,JDouble(2.0))))))

```

如果想执行`amountsJson.extract[List[Measure]]`来提取字段，直接执行是不行的，由于缺少`Measure`的构造方法，将会抛出

`org.json4s.package$MappingException: No constructor for type Measure, JObject(List((value,JDouble(2.0))))`

有一个方法可以解决是使用一个保存类型信息的合成字段。在json4s里，这样的字段被称为*类型提示*。默认情况下，键为`jsonClass`，值为

等于对应类型的名称。当值分解为JSON对象时，或者当从JSON中提取值时，会用类型提示来推断实际类型。为了启用这个功能，

可以配置并使用`withhints`方法，如下所示。

```scala
import org.json4s.Extraction.decompose
import org.json4s.jackson.JsonMethods.pretty

val hints = ShortTypeHints(List(
  classOf[Gram],
  classOf[Tablespoon],
  classOf[Teaspoon],
  classOf[Handful]))
// hints: org.json4s.ShortTypeHints = ShortTypeHints(List(class Gram, class Tablespoon, class Teaspoon, class Handful))

implicit val formats = DefaultFormats.withHints(hints)
val amountsJson = decompose(amounts)

pretty(amountsJson)
// String =
// [ {
//   "jsonClass" : "$read$Handful",
//   "value" : 2.0
// }, {
//   "jsonClass" : "$read$Gram",
//   "value" : 300.0
// }, {
//   "jsonClass" : "$read$Teaspoon",
//   "value" : 2.0
// } ]

amountsJson.extract[List[Measure]]
// List[Measure] = List(Handful(2.0), Gram(300.0), Teaspoon(2.0))


```



以下省略了自定义JSON解析时序列化方法，以及JSONP的介绍，如果想要了解可以参考原文。



