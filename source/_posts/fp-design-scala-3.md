# FPDIS 3. Type-Directed Programming

title: FPDIS 3.Type-Directed Programming
date: 2020-09-08 23:43:30
tags: [Scala, 读书笔记, Coursera] 
categories: [技术]
mathjax: true

------




这一小节都是阅读内容和测验



## Motivating Example & Type-Directed Programming

考虑一个排序函数

```scala
def sort(xs: List[Int]): List[Int] = {
  ...
  if (x < y)
  ...
}
```

这个排序只能实现对`Int`类型的排序，如果我们把它推广下

```scala
def sort[A](xs: List[A])(lessThan: (A, A) => Boolean): List[A] = {
  ...
  if (lessThan(x, y))
  ...
}
```

把比较函数作为第二个参数传入，就能实现通用的排序方法

其实Scala自带排序相关的类`Ordering`,上面的代码还能这样写

```scala
def sort[A](xs: List[A])(ord: Ordering[A]): List[A] = {
  ...
  ... if (ord.lt(x, y)) ...
  ...
}
```

但是这样还有个问题，就是每次都需要传入不同的`Ordering`

```scala
import scala.math.Ordering

sort(xs)(Ordering.Int)
sort(strings)(Ordering.String)
```

为了减少不必要的套版代码，可以使用隐式参数

```scala
def sort[A](xs: List[A])(implicit ord: Ordering[A]): List[A]
```

关于隐式参数的详细讲解，可以参考 *Programming in Scala*



习题需要升级才能提交，这里就不放出答案了。（下同）



所谓面向类型编程( **type-directed programming**) 能够从类型中推导出值来

编译器会在以下范围内搜索缺少的隐式类型

- 类型为 T
- 有implicit关键字
- 函数调用时可见, 或者在和T相关的伴生类中有定义

多个implciit可用时，如果优先级相等，会报`ambiguous implicit value`

但是内涵更多的implicit 定义优先级更高

- 具体类型大于泛型
- 子类大于父类

**Implicit Query**

```scala
scala> implicitly[Ordering[Int]]
res0: Ordering[Int] = scala.math.Ordering$Int$@73564ab0

# 对应的函数签名为
def implicitly[A](implicit value: A): A = value
```





## Type Classes

接着上面`Ordering`的例子

```scala
trait Ordering[A] {
  def compare(a1: A, a2: A): Int
}

object Ordering {
  implicit val Int: Ordering[Int] =
    new Ordering[Int] {
      def compare(x: Int, y: Int) = if (x < y) -1 else if (x > y) 1 else 0
    }
  implicit val String: Ordering[String] =
    new Ordering[String] {
      def compare(s: String, t: String) = s.compareTo(t)
  }
}

def sort[A: Ordering](xs: List[A]): List[A] = ...
```

这种情况下，我们称`Ordering`是一个`type class`（类型类）

`type class` 可以帮助我们实现多态

现在考虑实现一个分数类，并且给它也加上隐式的`Ordering`实现

```scala
/** A rational number
  * @param num   Numerator
  * @param denom Denominator
  */
case class Rational(num: Int, denom: Int)

object RationalOrdering {
  implicit val orderingRational: Ordering[Rational] =
    new Ordering[Rational] {
      def compare(q: Rational, r: Rational): Int =
        q.num * r.denom - r.num * q.denom
    }
}
```

对于`Ordering`这个隐式类来说，为了能使`sort`函数正常工作，它的实现应该满足如下性质

- 自反 compare(x, y) 和 compare(y, x)的符号应该相反
- 传递 如果 compare(x, y) < 0, compare(y, z) < 0 那么 compare(x, z) < 0
- 一致 如果 x, y 相等， 那么 compare(x, z) 与 compare(y, z) 的符号应该相同

这是wikipedia种对环(ring)的定义

> In mathematics, a **ring** is one of the fundamental algebraic structures used in abstract algebra. It consists of a set equipped with two binary operations that generalize the arithmetic operations of addition and multiplication. Through this generalization, theorems from arithmetic are extended to non-numerical objects such as polynomials, series, matrices and functions.
>
> This structure is so common that, by abstracting over the ring structure, developers could write programs that could then be applied to various domains 
>
> (arithmetic, polynomials, series, matrices and functions).                                                                    
>
>  ——https://en.wikipedia.org/wiki/Ring_(mathematics)



*环* <R, +, *>满足以下定律

- 是一个加法上的*交换群*
- 是一个乘法上的*Monid*
- 满足*乘法上的分配律*

下面一一解释其中的术语

在离散数学中，*群* G, 包含二元运算符`·`有这些性质

**封闭性** 


$$
  \forall a, b ∈ G, 有a · b \in G
$$

  

**结合律**

$$
  \forall a, b, c ∈ G, 有 (a · b) · c = a · (b · c)
$$



**有单位元**
$$
  \exists e ∈ G， 使得 \forall a∈G，都有 e · a = a · e = a
$$

**有逆元**

$$
  \forall a \in  G, \exists b \in G, 使得 a · b = b · a = e
$$



若一个群满足交换律
$$
  \forall a, b \in G, a · b = b · a
$$


则，群有被称为交换群(阿贝尔群/加法群)



Monid需要有如下性质

- 有单位元
- 满足结合律



乘法分配律好理解

- 左分配律 *a* ⋅ (*b* + *c*) = (*a* · *b*) + (*a* · *c*)
- 右分配律 (*b* + *c*) · *a* = (*b* · *a*) + (*c* · *a*) 



所有满足的性质可以看这张表（来自课程中的内容）

![](https://counter2015.com/picture/fpdis-3-1.jpg)

对于上述环，可以这样建模

```scala
trait Ring[A] {
  def plus(x: A, y: A): A
  def mult(x: A, y: A): A
  def inverse(x: A): A
  def zero: A
  def one: A
}

object Ring {
  implicit val ringInt: Ring[Int] = new Ring[Int] {
    def plus(x: Int, y: Int): Int = x + y
    def mult(x: Int, y: Int): Int = x * y
    def inverse(x: Int): Int = -x
    def zero: Int = 0
    def one: Int = 1
  }
}

// 用这个函数来检验加法的结合律
def plusAssociativity[A](x: A, y: A, z: A)(implicit ring: Ring[A]): Boolean =
  ring.plus(ring.plus(x, y), z) == ring.plus(x, ring.plus(y, z))
```

标准库的`type class Numeric`就满足环的性质

满足环性质的数据结构，可以方便地进行并行计算

## Conditional Implicit Definitions 

隐式参数是可以嵌套推导的

```scala
implicit def a: A = ...
implicit def aToB(implicit a: A): B = ...
implicit def bToC(implicit b: B): C = ...
implicit def cToD(implicit c: C): D = ... 
```

当我们用 *implicit query* 查询 `implicitly[D]`时

能链式的查到`implicit def a`

这样做的意义在后面编程作业的编写过程中，自然会发现。



当我们发现隐式参数可以连续推导时，自然很容易产生这么一个疑问？

可不可以递归地使用隐式定义，这样做有什么意义？

答曰：超纲，不在本课的讨论范围内。

## Implicit Conversions

隐式转换(Implicit Conversion)用来自动转换类型，来丰富API

考虑这么一个抽象语法树(Abstract Syntax Tree, AST)

```scala
sealed trait Json
case class JNumber(value: BigDecimal) extends Json
case class JString(value: String) extends Json
case class JBoolean(value: Boolean) extends Json
case class JArray(elems: List[Json]) extends Json
case class JObject(fields: (String, Json)*) extends Json
```

我们可以这样定义个json

```scala
JObject("name" -> JString("Paul"), "age" -> JNumber(42))
```

但其实这样并不简洁，最好是能达到 下面这样的结果

```scala
def obj(fields: (String, Json)*): Json = JObject(fields: _*)
obj("name" -> "Paul", "age" -> 42)
```

有一个办法是在伴生类中定义隐式转换

```scala
object Json {
  import scala.language.implicitConversions
  implicit def stringToJson(s: String): Json = JString(s)
  implicit def intToJson(n: Int): Json = JNumber(n)
  implicit def booleanToJson(b: Boolean): Json = JBoolean(b)
  implicit def arrayToJson(arr: List[Json]): Json = JArray(arr)
  implicit def objectToJson(o: Map[String, Json]): Json = JObject(o.toSeq: _*)
}
```

这样就简单了

```scala
scala> import Json._
import Json._

scala> obj("name" -> "Paul", "age" -> 42)
val res16: Json = JObject(ArraySeq((name,JString(Paul)), (age,JNumber(42))))
```

另外一个例子，是对方法做扩展

```scala
import java.util.concurrent.TimeUnit
case class Duration(value: Int, unit: TimeUnit)
```

我们希望做如下的简化

```scala
val delay = Duration(15, TimeUnit.SECONDS)

// val delay = 15.seconds
```



这样只要引入`implicit class`就行

```scala
case class Duration(value: Int, unit: TimeUnit)
object Duration {
  object Syntax {
    import scala.language.implicitConversions
    implicit class HasSeconds(n: Int) {
      def seconds: Duration = Duration(n, TimeUnit.SECONDS)
    }
  }
}

import Duration.Syntax._
14.seconds
```

`14.seconds` 会被编译成`new HasSeconds(15).seconds`

编译器会在以下情况下寻找隐式转换

- T不符合表达式的预期类型
- 在选择e.m中，如果成员m在T上不可访问
- 在选择e.m（args）中，如果成员m在T上可访问，但不适用于参数args

使用隐式转换时要特别谨慎，因为会把一个类型自动转成另一个类型，滥用的话可能导致用户抓狂

调试隐式转换时，scalac的选项`-Xprint:typer`可以让我们看到经过编译器转换隐式推断后的代码

隐式转换是一个`核武器`，不是什么时候都用它，能用更小的代价优化代码，就不要滥用隐式转换，过多的隐式转换会导致代码繁杂。

## 编程作业： Json编码解码器



### 问题背景

本次作业的目标是实现一个Type Directed 的 序列化库

它使用面向类型的操作，组合简单类型值的序列化器，为更复杂类型的值构造序列化器。

Json的数据结构定义如下

```scala
sealed trait Json

object Json {

  /** The JSON `null` value */

  case object Null extends Json

  /** JSON boolean values */

  case class Bool(value: Boolean) extends Json

  /** JSON numeric values */

  case class Num(value: BigDecimal) extends Json

  /** JSON string values */

  case class Str(value: String) extends Json

  /** JSON objects */

  case class Obj(fields: Map[String, Json]) extends Json

  /** JSON arrays */

  case class Arr(items: List[Json]) extends Json

}
```



举个具体的例子，对于一个Json字符串数据

```json
{
  "foo": 0,
  "bar": [true, false]
}
```

在代码中的表现为

```scala
Json.Obj(Map(
  "foo" -> Json.Num(0),
  "bar" -> Json.Arr(Json.Bool(true), Json.Bool(false))
))
```



`Encoder`做的工作，就是把不同类型的数据转换成`Json class`的类型

其定义签名如下

```scala
trait Encoder[-A] {
  def encode(value: A): Json
}
```

这里的`-A`不知道还有印象没有，这是代表`逆变`的意思

即 如果 `Arr` 是 `Json`类型的子类，那么 `Encoder[Arr]` 是 `Encoder[Json]`的父类

依据[里氏替换原则](https://zh.wikipedia.org/zh-hans/%E9%87%8C%E6%B0%8F%E6%9B%BF%E6%8D%A2%E5%8E%9F%E5%88%99)，父类可以被等价地替换为子类

这意味着，程序中所有出现`Encoder[Arr]`的地方，都能被替换成`Encoder[Json]`

更多关于Variances（型变/逆变/不变）的说明可以参考[官方文档](https://docs.scala-lang.org/tour/variances.html)





与其他具体的 Json 值(`Num`, `Str`)不同，Json `Object` 比较特殊，可以将两个`Object`组合起来

以构建一个更大的 Json `Object`，其中包含两个对象的所有字段。

在名为ObjectEncoder的Encoder子类中`zip`定义其实现



```scala
/**
  * A specialization of `Encoder` that returns JSON objects only
  */
trait ObjectEncoder[-A] extends Encoder[A] {
  // Refines the encoding result to `Json.Obj`
  def encode(value: A): Json.Obj

  /**
    * Combines `this` encoder with `that` encoder.
    * Returns an encoder producing a JSON object containing both
    * fields of `this` encoder and fields of `that` encoder.
    */
  def zip[B](that: ObjectEncoder[B]): ObjectEncoder[(A, B)] =
    ObjectEncoder.fromFunction { case (a, b) =>
      Json.Obj(this.encode(a).fields ++ that.encode(b).fields)
    }
}
```



相对地，**Decoder[A]**将Json类型的值转换为**A**类型的值：

```scala
trait Decoder[+A] {

  /**
    * @param data The data to de-serialize
    * @return The decoded value wrapped in `Some`, or `None` if decoding 
        failed
    */
  def decode(data: Json): Option[A]
}
```

这里使用`Option[A]`而不是`A`作为返回类型，是因为在 Json 数据的解析过程中，可能会出现失败的情况

这里提供了一些工具类，就不赘述了



### Our Tasks

我们的任务，是从简单的基本类型开始，构建 Json 解析器的实例，直到能实现对 case class 的解析为止

简单的验证工作可以用`sbt console`来完成，但要注意的是修改代码后需要重启REPL才能更新



首先要实现的是

- Encoder[String], Encoder[Boolean]
- Decoder[Int], Decoder[String], Decoder[Boolean]

可以复用`Encoder.fromFunction`， `Decoder.fromFunction`， `Decoder.fromPartialFunction`的逻辑

同时需要确保`Docoder[Int]` 能拒绝浮点数类型的输入

下面是一个简单的实现

```scala
/** An encoder for `String` values */
implicit val stringEncoder: Encoder[String] =
  Encoder.fromFunction(Json.Str)

/** An encoder for `Boolean` values */
implicit val booleanEncoder: Encoder[Boolean] =
  Encoder.fromFunction(Json.Bool)

/** A decoder for `Int` values. Hint: use the `isValidInt` method of `BigDecimal`. */
implicit val intDecoder: Decoder[Int] =
  Decoder.fromPartialFunction{ case Json.Num(x) if x.isValidInt => x.toInt }

/** A decoder for `String` values */
implicit val strDecoder: Decoder[String] =
  Decoder.fromPartialFunction{case Json.Str(s) => s}

/** A decoder for `Boolean` values */
implicit val booleanDecoder: Decoder[Boolean] =
  Decoder.fromPartialFunction{case Json.Bool(b) => b}
```



基本类型的编解码函数实现后，需要实现的是集合`List`类型的函数,已经给出了`Encoder`部分的实现

也可以简单地验证

```scala
implicit def listEncoder[A](implicit encoder: Encoder[A]): Encoder[List[A]] =
  Encoder.fromFunction(as => Json.Arr(as.map(encoder.encode)))

scala> implicitly[Encoder[List[Int]]]
res14: codecs.Encoder[List[Int]] = codecs.Encoder$$anon$1@2e65d7ca
```

照猫画虎，可以实现`Decoder`

```scala
/**
  * A decoder for JSON arrays. It decodes each item of the array
  * using the given `decoder`. The resulting decoder succeeds only
  * if all the JSON array items are successfully decoded.
*/
implicit def listDecoder[A](implicit decoder: Decoder[A]): Decoder[List[A]] =
  Decoder.fromPartialFunction{
    case Json.Arr(vs) => vs.flatMap(decoder.decode)
}
```



实现后也可以在REPL中测试

```scala
scala> import codecs._; import Util._
import codecs._
import Util._

scala> val a = parseJson("""["a", "b"]""")
a: Option[codecs.Json] = Some(Arr(List(Str(a), Str(b))))

scala> a.flatMap(_.decodeAs[Int])
res1: Option[Int] = None

scala> a.flatMap(_.decodeAs[List[String]])
res2: Option[List[String]] = Some(List(a, b))

scala> a.flatMap(_.decodeAs[List[Int]])
res3: Option[List[Int]] = Some(List())
```

我们来看`ObjectEncoder`的例子

```scala
scala> val xField = ObjectEncoder.field[Int]("x")
xField: codecs.ObjectEncoder[Int] = codecs.ObjectEncoder$$anon$2@527fbd73

scala>  xField.encode(42)
res4: codecs.Json.Obj = Obj(Map(x -> Num(42)))
```

对于多个字段，可以这样实现

```scala
scala> val pointEncoder: ObjectEncoder[(Int, Int)] = {
     | 
     |   val xField = ObjectEncoder.field[Int]("x")
     | 
     |   val yField = ObjectEncoder.field[Int]("y")
     | 
     |   xField.zip(yField)
     | 
     | }
pointEncoder: codecs.ObjectEncoder[(Int, Int)] = codecs.ObjectEncoder$$anon$2@3451467e

scala> pointEncoder.encode((4,5))
res8: codecs.Json.Obj = Obj(Map(x -> Num(4), y -> Num(5)))



```

依据`ObjectEncoder.field` 实现 `Decoder.field`就简单了

```scala
// ObjectEncoder的实现
/**
  * An encoder for values of type `A` that produces a JSON object with one field
  * named according to the supplied `name` and containing the encoded value.
  */
def field[A](name: String)(implicit encoder: Encoder[A]): ObjectEncoder[A] =
  ObjectEncoder.fromFunction(a => Json.Obj(Map(name -> encoder.encode(a))))

// Decoder的实现
/**
  * A decoder for JSON objects. It decodes the value of a field of
  * the supplied `name` using the given `decoder`.
  */
def field[A](name: String)(implicit decoder: Decoder[A]): Decoder[A] =
  Decoder.fromFunction {
    case Json.Obj(fs) => decoder.decode(fs(name))
  }
```

接下来我们需要实现`case class`的编码/解码工作

当前已经有`Encoder[Person]`的实现代码了，现在需要实现的是

- `Decoder[Person]`
- `Encoder[Contacts]`, `Decoder[Contacts]`

```scala
case class Person(name: String, age: Int)

object Person extends PersonCodecs

trait PersonCodecs {

  /** The encoder for `Person` */
  implicit val personEncoder: Encoder[Person] =
    ObjectEncoder.field[String]("name")
      .zip(ObjectEncoder.field[Int]("age"))
      .transform[Person](user => (user.name, user.age))

  /** The corresponding decoder for `Person` */
  implicit val personDecoder: Decoder[Person] =
    ???

}
```



有了之前实现多个`Decoder`的经验，不难想到，我们需要的是使用`Decoder.field`提取出每一个`Person`中的每一个字段，使用`zip`组合，再用`transform`转换成需要的类

```scala
implicit val personDecoder: Decoder[Person] = {
  Decoder.field[String]("name")
    .zip(Decoder.field[Int]("age"))
    .transform[Person]{case (name, age) => Person(name, age)}
}
```

写完后可以简单测试下

```scala
scala> import codecs._
import codecs._

scala> import Util._
import Util._

scala> val maybeJsonObj    = parseJson(""" { "name": "Alice", "age": 42 } """)
maybeJsonObj: Option[codecs.Json] = Some(Obj(Map(name -> Str(Alice), age -> Num(42))))

scala> println(maybeJsonObj.flatMap(_.decodeAs[Person]))
Some(Person(Alice,42))

scala> val maybeJsonObj2   = parseJson(""" { "name": "Alice", "age": "42" } """)
maybeJsonObj2: Option[codecs.Json] = Some(Obj(Map(name -> Str(Alice), age -> Str(42))))

scala> println(maybeJsonObj2.flatMap(_.decodeAs[Person]))
None

scala>  println(renderJson(Person("Bob", 66)))
{"name":"Bob","age":66}
```



至于`Encoder[Contacts]`和`Decoder[Contacts]`的实现，用之前的`listEncoder`和`listDecoder`就行

下面把需要用到的代码放一起方便查看

```scala
implicit def listDecoder[A](implicit decoder: Decoder[A]): Decoder[List[A]] =
  Decoder.fromPartialFunction{
    case Json.Arr(vs) => vs.flatMap(decoder.decode)
  }

implicit def listEncoder[A](implicit encoder: Encoder[A]): Encoder[List[A]] =
  Encoder.fromFunction(as => Json.Arr(as.map(encoder.encode)))

case class Contacts(people: List[Person])

object Contacts extends ContactsCodecs

trait ContactsCodecs {

  // TODO Define the encoder and the decoder for `Contacts`
  // The JSON representation of a value of type `Contacts` should be
  // a JSON object with a single field named “people” containing an
  // array of values of type `Person` (reuse the `Person` codecs)

}
```

下面是一个实现，目前是依靠IDE的提示对类型配平做出来的

<del>为什么这样实现，你问我，我问谁</del>

```scala
trait ContactsCodecs {

  // The JSON representation of a value of type `Contacts` should be
  // a JSON object with a single field named “people” containing an
  // array of values of type `Person` (reuse the `Person` codecs)
  implicit val contactsEncoder: Encoder[Contacts] =
    ObjectEncoder.field[List[Person]]("people").transform[Contacts]{ case Contacts(ps) =>  ps}

  implicit val contactsDecoder: Decoder[Contacts] =
    Decoder.field[List[Person]]("people").transform[Contacts](ps => Contacts(ps))

}
```



### 附加题



- 你能显示的写出编译器推断`Encoder[List[Person]]`的结果吗

  ```
  Encoder[List[Person]]
  = Encoder[listEncoder[Person](implicit encoder: Encoder[Person])]
  = Encoder[listEncoder[Person](implicit encoder: Encoder[Person])]
  = Encoder[listEncoder[Person](
    ObjectEncoder.field[String]("name")(Encoder[String])
        .zip(ObjectEncoder.field[Int]("age"))(Encoder[Int])
        .transform[Person](user => (user.name, user.age))
  )
  = 下面就不展开了（要吐了，编译器也太强了
  ```

  

- 可以对`Option`的字段定义编码/解码器吗

  由下一个问题，知可以

- 可不可以对类型`A`定义一个隐式实例`Encoder[Option[A]]`（就像`Encoder[List[A]]`一样）

  蹩脚地模仿着实现

  ```scala
  implicit def optionEncoder[A](implicit encoder: Encoder[A]): Encoder[Option[A]] =
    Encoder.fromFunction{
      case Some(e) => encoder.encode(e)
      case _ => Json.Null
    }
  ```

  

- 除了`case class`，能不能对`sealed trait`也定义编解器

  这样问了，肯定是可以，比如https://github.com/circe/circe-derivation/issues/8

  我还没弄明白怎么实现的。