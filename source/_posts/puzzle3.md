#  Scala Puzzle 3.Location, Location, Location

title:  Scala Puzzle 3.Location, Location, Location
date: 2019-02-23 14:27:45
tags:  [scala, 读书笔记]
categories: [puzzles]

------


官网地址：http://scalapuzzlers.com/#pzzlr-004

## 3.Location, Location, Location

Scala为我们提供了许多简洁的特性，在许多面向对象的语言中，类构造器(class constructor)需要接受参数以赋值给类成员(class member)。

```java
class MyClass(param 1, param2, ...) {
  val member1 = param1
  val member2 = param2
  ...
}
```

Scala代码允许你一口气声明成员变量和构造器参数。

```scala
class MyClass(val member1 = "default_value1", val member2, ...){
  ...
}
```



一个简单的例子

```scala
scala> class B(val p: String = "World"){
     |   val m = p
     |   println(m)
     | }
defined class B


scala> new B
World
res17: B = B@6134ac4a

scala> new B("Hello")
Hello
res18: B = B@5f574cc2

```

再深入一些，我们来观察以下代码的结果

```scala
scala> trait A {
     |   val audience: String
     |   println("Hello " + audience)
     | }
defined trait A

scala> class BMember(val a: String = "World") extends A {
     |   val audience: String = a
     |   println("I repeat: Hello " + audience)
     | }
defined class BMember

scala> class BConstructor(val audience: String = "World") extends A {
     |   println("I repeat: Hello " + audience)
     | }
defined class BConstructor

scala> new BMember("Readers")
Hello null
I repeat: Hello Readers
res2: BMember = BMember@239b0f9d

scala> new BConstructor("Readers")
Hello Readers
I repeat: Hello Readers
res3: BConstructor = BConstructor@40e10ff8
```



## 解释

这里涉及到几个关键点：

- “Readers”赋值给`audience`是什么时刻可见(becomes visible)的
- 参数缺省值"World“在这种情况下是否会影响`audience`的取值
- `audience`的成员声明移动至参数列表是否会影响程序



从输出的值来看，当构造器的参数设定为"Readers"时，默认的参数值"World"不会影响`audience`的取值。

剩下两个问题需要我们进一步地观察代码。

```scala
class BMember(val a: String = "World") extends A {
  val audience: String = a
  println("I repeat: Hello " + audience)
}

class BConstructor(val audience: String = "World") extends A {
      println("I repeat: Hello " + audience)
}
```

这两个类的声明都可以归纳成以下形式：

`class c(params) extends superclass { statements}`

对应的初始化语句`new BMember("Readers")`和`new BConstructor("Readers")`执行顺序:

1. 求出`params`的值，这里是单纯的字符串"Readers"

2. 执行`superclass {statements}`

   - 首先执行`superclass`部分，对于这个例子，就是A的构造函数

     ```scala
     trait A {
       val audience: String
       println("Hello " + audience)
     }
     ```

   - 接下来执行`{statements}`部分，也就是子类`BMember`或`BConstructor`中的语句



这里我们省略了`trait`的特性信息，你可以单纯地把它当作一个父类。

对于`BMemeber`来说，首先将"Readers"赋值给其构造器参数`a`，接着执行`A`的构造器参数,此时`A`中`audience`还未初始化，所以执行语句` println("Hello " + audience)`输出结果为"Hello null"。这里执行完后程序回到`BMember`内部的代码块，此时将构造器参数`a`赋值给成员变量audience,所以执行语句` println("I repeat: Hello " + audience)`输出结果为"I repeat: Hello Readers"



对于`BConstructor`来说情况略有不同，作为构造器参数求值的一部分，构造器参数"Readers"被立刻赋值给成员变量`audience`,所以执行到`A`中的代码块中时`audience`就不再是null了。

这两种写法中，第二种显然出错的机会更少。

对于这个问题，[scala官方文档](https://docs.scala-lang.org/tutorials/FAQ/initialization-order.html)解释得不错。

附上一个详细的例子说明

```scala
scala> trait A {
     |   val audience: String
     |   println("Hello " + audience)
     | }
defined trait A

scala> trait AfterA {
     |   val introduction: String
     |   println(introduction)
     | }
defined trait AfterA

scala> class BEvery(val audience: String) extends {
     |   val introduction = {
     |     println("Evaluating early def"); "Are you there?" }
     | } with A with AfterA {
     |   println("I repeat: Hello " + audience)
     | }
defined class BEvery

scala> new BEvery({ println("Evaluating param"); "Readers" })
Evaluating param
Evaluating early def
Hello Readers
Are you there?
I repeat: Hello Readers
res6: BEvery = BEvery@6ea1bcdc

```



## 小结

对于有继承关系的类，特别注意其初始化可能导致的问题。



## 参考资料

1. [In Scala, what is an “early initializer”?](https://stackoverflow.com/questions/4712468/in-scala-what-is-an-early-initializer)
2. [Scala - initialization order of vals](https://stackoverflow.com/questions/14568049/scala-initialization-order-of-vals)
3. [Getting a null with a val depending on abstract def in a trait](https://stackoverflow.com/questions/29673758/getting-a-null-with-a-val-depending-on-abstract-def-in-a-trait?noredirect=1&lq=1)

