#  Scala Puzzle 4.Now You See Me, Now You Don't

title:  Scala Puzzle 4.Now You See Me, Now You Don't
date: 2019-03-24 17:10:25
tags:  [scala, 读书笔记]
categories: [puzzles]

------



官网地址：<http://scalapuzzlers.com/#pzzlr-005>

## 问题的提出

多重继承过程中的初始化过程会将事情带入到预想不到的地步，

下面这段程序演示了这个例子

``` scala
trait A {
    val foo: Int
    val bar = 10
    println("In A: foo: " + foo + ", bar: " + bar)
}

class B extends A {
    val foo: Int = 25
    println("In B: foo: " + foo + ", bar: " + bar)
}

class C extends B {
    override val bar = 99
    println("In C: foo: " + foo + ", bar: " + bar)
}
```



```scala
scala> new C
In A: foo: 0, bar: 0
In B: foo: 25, bar: 0
In C: foo: 25, bar: 99
res0: C = C@58fe0499

scala> new B
In A: foo: 0, bar: 10
In B: foo: 25, bar: 10
res1: B = B@1698ee84


```

## 分析

要理解这个问题，我们得分析观察程序的每一步是如何执行的。

Scala类都有一个非显示定义，但是在类定义过程中始终存在的主构造器，类定义中的所有语句(包括字段定义)形成了主构造器的程序体。

因此，A,B,C中所有的代码都属于一个构造器程序体。

在不存在预先定义(early definitions)的情况下，通常意义上的`val`字段的初始化顺序遵循以下规则：

1. 父类在子类前初始化
2. 在1的前提下，按照声明的顺序初始化

当`val`字段被覆写(override)时，并不会被多次初始化，因此，虽然上述代码中的`bar`字段看起来在每个字段中都有定义，但是却并非如此，由于以下两个约束：

1. 当一个`val`字段被覆写时，只能初始化一次
2. 覆写的`val`字段在父类构造期间会有一个缺省的初始值



### 缺省初始值

Scala的缺省初始值为:

- Byte、Short和Int          =>    0
- Long                             =>    0L
- Float                             =>    0.0f
- Double                          =>    0.0d
- Char                              =>   '\0'
- Boolean                         =>   false
- Unit                                =>   ()
- 其他                               =>   null

有了这些知识，再来解释之前代码的输出就水到渠成了。

虽然表面上看，在trait A和class B中给`bar`分配了一个初始值10，然而并非如此，因为在子类C构造时，该字段就已经被初始化，而覆写的`val`字段不能被再次初始化，此时A,B中的`bar`便使用的是缺省的初始值0而不是10。由于初始化顺序，class C覆写了bar,并将其赋值为99，所以trait A对bar的赋值是不可见的。

`foo`的值也是类似的问题，因为class B中给`foo`赋值为25，其父类A不可见所以值为0，而子类可见所以是25，B的构造过程分析和上述过程类似，这里就不赘述了。



## 解决方法

### Methods

一个简单的方法是将`bar`的沈明从`val`改为`def`

```scala
trait A {
  val foo: Int
  def bar: Int = 10
  println("In A: foo: " + foo + ", bar: " + bar)
}

class B extends A {
  val foo: Int = 25
  println("In B: foo: " + foo + ", bar: " + bar)
}

class C extends B {
  override def bar: Int = 99
  println("In C: foo: " + foo + ", bar: " + bar)
}


scala> new C
In A: foo: 0, bar: 99
In B: foo: 25, bar: 99
In C: foo: 25, bar: 99
res3: C = C@2f9a01c1



```



这种方法之所以有效是因为`def`这个方法的内容不属于主构造器，因此不参与类的初始化过程，此外，由于Class C中覆写了`bar`，所以会使用这个覆写过的定义。因此，三个地方的`bar`都使用的是99这个值。

这个方法其实并不理想，甚至可以说“脏“，一方面，每次调用`bar`时都需要额外的计算开销，另一方面，Scala遵循统一访问原则，所以在父类中定义的参数方法，可以在子类中将其覆写为一个`val`字段，这会导致类似的情况再次出现。



### LAZY VALS

`lazy val` 和`val`的区别在于，前者是在首次访问时初始化，而后者是在定义时初始化的

```scala
trait A {
  val foo: Int
  lazy val bar = 10
  println("In A: foo: " + foo + ", bar: " + bar)
}

class B extends A {
  val foo: Int = 25
  println("In B: foo: " + foo + ", bar: " + bar)
}

class C extends B {
  override lazy val bar = 99
  println("In C: foo: " + foo + ", bar: " + bar)
}

scala> new C
In A: foo: 0, bar: 99
In B: foo: 25, bar: 99
In C: foo: 25, bar: 99
res5: C = C@533b266e


```



将`bar`声明为`lazy val`，意味着在trait A构造期间就初始化了99——因为这是它首次被访问的地方。

但是这种方式也有缺点：

- 无法声明抽象的`lazy val`
- 循环引用可能导致死锁和爆栈
- 引入轻微的性能成本

如果你无论如何都想声明抽象的`lazy val`的话，有如下的权宜之计

1. Declare an abstract strict val, and hope subclasses will implement it as a lazy val or with an early definition. If they do not, it will appear to be uninitialized at some points during construction.
2. Declare an abstract def, and hope subclasses will implement it as a lazy val. If they do not, it will be re-evaluated on every access.
3. Declare a concrete lazy val which throws an exception, and hope subclasses override it. If they do not, it will… throw an exception.



### EARLY DEFINITIONS

```scala
trait A {
  val foo: Int
  val bar = 10
  println("In A: foo: " + foo + ", bar: " + bar)
}

class B extends A {
  val foo: Int = 25
  println("In B: foo: " + foo + ", bar: " + bar)
}

class C extends {
  override val bar = 99
} with B {
  println("In C: foo: " + foo + ", bar: " + bar)
}

scala> new C
In A: foo: 0, bar: 99
In B: foo: 25, bar: 99
In C: foo: 25, bar: 99
res7: C = C@578524c3

```

还记得[前一章节](http://counter2015.com/2019/02/21/puzzle3/)中最后出现的例子吗，也是用的`early definitions`的方法来定义字段的，预先定义的代码会在主构造器之前运行，这样可以确保`bar`在trait A前被初始化成99



## 练习

```scala
abstract class A {
  val x1: String
  val x2: String = "mom"

  println("A: " + x1 + ", " + x2)
}
class B extends A {
  val x1: String = "hello"
  final val x3 = "goodbye"

  println("B: " + x1 + ", " + x2)
}
class C extends B {
  override val x2: String = "dad"

  println("C: " + x1 + ", " + x2)
}
abstract class D {
  val c: C
  val x3 = c.x3   
  println("D: " + c + " but " + x3)
}
class E extends D {
  val c = new C
  println(s"E: ${c.x1}, ${c.x2}, and $x3...")
}
```

问：`new E`的输出结果是什么？
答案下方点击即可查看，如果不懂的话，可以重看下前一章关于初始化执行顺序的讲解。

{% 
spoiler scala> new E<br>
D: null but goodbye<br>
A: null, null<br>
B: hello, null<br>
C: hello, dad<br>
E: hello, dad, and goodbye...  %}



## 参考资料

[Why is my abstract or overridden val null](https://docs.scala-lang.org/tutorials/FAQ/initialization-order.html)