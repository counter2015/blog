# Progfun 4. Types and Pattern Matching

title: Progfun 4.Types and Pattern Matching
date: 2019-09-24 02:22:30
tags: [scala, 读书笔记, Coursera] 
description:  长文预警，预计阅读时间：1小时。
categories: [技术]

------



### 4.1 Objects Everywhere

基本类型在scala里也是对象，下面是`Boolean`的一个实现方式

```scala
abstract class Boolean {
  def ifThenElse[T](t: => T, e: => T): T
  
  def &&(x: => Boolean): Boolean = ifThenElse(x, false)
  def ||(x: => Boolean): Boolean = ifThenElse(true, x)
  
  def unary_! : Boolean = ifThenElse(False, True)
  
  def ==(x: Boolean): Boolean = ifThenElse(x, x.unary_!)
  def !=(x: Boolean): Boolean = ifThenElse(x.unary_!, x)
}

object true extends Boolean {
  def ifThenElse[T](t: => T, e: => T) = t
}

object false extends Boolean {
  def ifThenElse[T](t: => T, e: => T) = e
}
```

上面这段代码看起来很难理解，我尝试在REPL里编写，发现无法被正确地编译。

后来在stackoverflow上找到了[解释](https://stackoverflow.com/questions/16064384/scala-booleans-code-snippet),`true/false`为保留字，无法被重新定义，所以换一个变量名称就行了。

修改`true/fasle`为`True/False`即可，如下所示。



```scala
abstract class Boolean {
  def ifThenElse[T](t: => T, e: => T): T
  
  def &&(x: => Boolean): Boolean = ifThenElse(x, False)
  def ||(x: => Boolean): Boolean = ifThenElse(True, x)
  
  def unary_! : Boolean = ifThenElse(False, True)
  
  def ==(x: Boolean): Boolean = ifThenElse(x, x.unary_!)
  def !=(x: Boolean): Boolean = ifThenElse(x.unary_!, x)
}

object True extends Boolean {
  def ifThenElse[T](t: => T, e: => T) = t
}

object False extends Boolean {
  def ifThenElse[T](t: => T, e: => T) = e
}
```

这里用到了`Call by name`和`unary_`定义一元运算符的知识, 以`!=`为例讲解下实现

```scala
def !=(x: Boolean): Boolean = ifThenElse(x.unary_!, x)
```



假如是 `True.!=(x)` , 我们期望的结果和x的取值相反（`True != True 返回Fasle`,` True != False 返回 True`），而`True.ifThenElse(t, e)`总是会返回`t`, 此时`t`应该为`x.unary_!`即`!x` 。假如是`False.!=(x)`,我们期望的结果与x的取值相同, 而`False.ifThenElse(t, e)`总是会返回`e`,此时`e`应该为`x` 结合`ifThenElse`的特点，`!=`的实现就像上面这样子了（不明白的话再重新读一遍这段话）

```scala
scala> !True
res6: Boolean = False$@2449cff7

scala> True || False
res8: Boolean = True$@13006998

scala> False || True
res9: Boolean = True$@13006998

scala> False && True
res10: Boolean = False$@2449cff7
```



然后Martin布置的一个习题是实现`<`运算符

- 假设 False < True

其实这里相当于给出了一个真值表。

当True调用`<`方法时，返回的结果需要为False

当False < False 返回 False

(也许你会看的有点晕，但注意上面的`True/False`不是值，而是我们自定义的`object`)

考虑利用`ifThenElse`来实现

而`True.ifThenElse` 总是会返回`t`, 我们需要`t`为Fasle

`Flase.ifThenElse`总是会返回`e`,我们希望返回的值和右边相同

因而写出如下代码。



```scala
abstract class Boolean {
  def ifThenElse[T](t: => T, e: => T): T
  
  def &&(x: => Boolean): Boolean = ifThenElse(x, False)
  def ||(x: => Boolean): Boolean = ifThenElse(True, x)
  
  def unary_! : Boolean = ifThenElse(False, True)
  
  def ==(x: Boolean): Boolean = ifThenElse(x, x.unary_!)
  def !=(x: Boolean): Boolean = ifThenElse(x.unary_!, x)
  
  def <(x: Boolean): Boolean = ifThenElse(False, x)
}

object True extends Boolean {
  def ifThenElse[T](t: => T, e: => T) = t
}

object False extends Boolean {
  def ifThenElse[T](t: => T, e: => T) = e
}
```

验证结果如下

```scala
scala> False < True
res2: Boolean = True$@13006998

scala> False < False
res3: Boolean = False$@2449cff7

scala> True < False
res4: Boolean = False$@2449cff7

scala> True < True
res5: Boolean = False$@2449cff7
```







给出下列函数前面，要求完成剩余部分，实现自然数的抽象。

```scala
abstract class Nat {
  def isZero: Boolean 
  def predecessor: Nat
  def sucessor: Nat
  def +(that: Nat): Nat
  def -(that: Nat): Nat
}

Object Zero extends Nat
class Succ(n: Nat) extends Nat
```

实现过程如下(不要忘记重新开启一个REPL，不然`Boolean`的定义就是使用的上面自己实现的了)

```scala
abstract class Nat {
  def isZero: Boolean 
  def predecessor: Nat
  def sucessor: Nat = new Succ(this)
  def +(that: Nat): Nat
  def -(that: Nat): Nat
}

object Zero extends Nat {
	def isZero = true
  def predecessor = throw new Error("Zero has no predecessor")
  def +(that: Nat): Nat = that
  def -(that: Nat): Nat = if (that.isZero) this else throw new Error("Negative number")
}

class Succ(n: Nat) extends Nat {
  def isZero = false
  def predecessor = n
  def +(that: Nat): Nat = new Succ(n+that)
  def -(that: Nat): Nat = if(that.isZero) this else n - that.predecessor
}
```

这里`Succ`的`+`,`-`的实现可能会让人迷惑，但是这个递归函数是能正常结束的

具体来说，对于`def +(that: Nat): Nat = new Succ(n+that)`, 调用时外面多套了一次`Succ`，加号左边的数减了一，最后会变成`Succ(Succ...(Succ(Zero.+(that))...)`

减号同理

下面是1 + 1 = 2的验证

```scala
scala> val x = new Succ(Zero)
x: Succ = Succ@6b44435b

scala> val y = new Succ(Zero)
y: Succ = Succ@4da602fc

scala> val z = x + y
z: Nat = Succ@5b07730f

scala> z.predecessor.predecessor
res2: Nat = Zero$@69da0b12
```



### 4.2 Functions as Objects

一个匿名函数的展开求值

`(x: Int) => x * x`  等价于 

```scala
{
  class AnoFun extends Function1[Int, Int] {
    def apply(x: Int) = x * x
  }
  new AnoFun
}
```

可以简化为

```scala
new Function1[Int, Int]{ 
  def apply(x: Int) = x * x
}
```

当我们调用函数`f(a,b)`时，其实可以展开为`f.apply(a,b)`

```scala
val f = (x: Int) => x * x
f(7)
```

上面的代码等价于

```scala
val f = new Function1[Int, Int]{
  def apply(x: Int) = x * x
}
f.apply(7)
```

你可能好奇Fucuntion后面的编号有多少个

```scala
scala> new Function
Function    Function10   Function13   Function16   Function19   Function21   Function4   Function7   FunctionalInterface   
Function0   Function11   Function14   Function17   Function2    Function22   Function5   Function8                         
Function1   Function12   Function15   Function18   Function20   Function3    Function6   Function9               
```

很明显，22个<del>但是为什么是22，42不好吗</del>。参见官方文档对于[Function22](https://www.scala-lang.org/api/2.13.0/scala/Function22.html)的解释(这里会涉及到协变和逆变暂且不表)。

关于这点有些文章有讨论过

- [Scala and 22](https://underscore.io/blog/posts/2016/10/11/twenty-two.html)

- [Why are scala functions limited to 22 parameters?](https://stackoverflow.com/questions/4152223/why-are-scala-functions-limited-to-22-parameters)



下一个练习，给出如下的`List`定义，实现

- List()
- List(1)
- List(2,3)

这样的初始化

```scala
trait List[T] {
  def isEmpty: Boolean 
  def head: T
  def tail: List[T]
}

class Cons[T](val head: T, val tail: List[T]) extends List[T] {
  def isEmpty = false
}

class Nil[T] extends List[T] {
  def isEmpty = true
  def head = throw new NoSuchElementException("Nil.head")
  def tail = throw new NoSuchElementException("Nil.tail")
}
```

实现如下

```scala
object list {
  def apply[T](x1: T, x2: T):List[T] =  new Cons(x1, new Cons(x2, new Nil))
  def apply[T](x: T): List[T] = new Cons(x, new Nil)
  def apply[T]() = new Nil
}
```

验证如下

```scala
scala> list()
res13: Nil[Nothing] = Nil@427475a2

scala> list(2)
res14: List[Int] = Cons@3833a045

scala> list(2,23)
res15: List[Int] = Cons@5ec80aad
```



### 4.3 Subtyping and Generics

**Type bounds**

这个名词为了避免引起谬误，就不翻译了。为了解释这个概念，下面用一个例子来讲解说明。

假设我们有这么个函数`def assertAllPos(s: IntSet): IntSet`

- 如果`IntSet`里的所有元素都为正数，返回`IntSet`本身
- 否则抛出异常



在大多数情况下都能正常工作，但是考虑到如下条件：

比如 

`def assertAllPos(s: Empty): Empty = s`，希望在空集时直接返回本身

`def assertAllPos(s: NoEmpty): NoEmpty`希望在非空集的时候按照之前的逻辑来判断



这里我们需要一个更加**通用**的定义来描述这一行为，我们可以把`EmptySet`和`NoEmptySet`都视作`IntSet`的子集

于是就引出了上文所述的概念`Type bounds`

`def assertAllPos[s <: IntSet](r: S): S = ???`

这里`s <: IntSet`表明了`s`的类型是`IntSet`的子类型， 可以说`IntSet`是`s`类型的上界

- S <: T  S是T的子类型(subtype)
- S >: T  S是T的父类型(supertype)

这也很好记忆，大于号小于号标识类型的大小关系，冒号标记该参数的类型

当写下`S >: NonEmpty`时，这代表`S`可以取的类型有

- NonEmpty
- IntSet
- AnyRef
- Any

当然也可以连起来用上界下界

```scala
[S >: NonEmpty <: IntSet]
```

这严格定义了`S`的类型处于`NonEmpty`和`IntSet`之间

嗯，接下来会引入一个不那么好理解的概念 协变（Convariance）

我们知道

`NonEmpty <: IntSet`

那么能否推断出

`List[NonEmpty] <： List[IntSet]`

答案是“Yes”

你可以参照官方文档中关于[协变的部分](https://docs.scala-lang.org/tour/variances.html)了解更多的细节，或者，直接查看Scala对`List`的实现

```scala
type List[+A] = scala.collection.immutable.List[A]
```

看到这一层就可以了，我们注意到，定义`List`时有一个泛型`A`，同时还有一个加号“+”。

> A type parameter `A` of a generic class can be made covariant by using the annotation `+A`. For some `class List[+A]`, making `A` covariant implies that for two types `A` and `B` where `A` is a subtype of `B`, then `List[A]` is a subtype of `List[B]`. This allows us to make very useful and intuitive subtyping relationships using generics.



简单来说，当泛型中有加号标识时，说明这个类型参数是协变的，对于上面的例子来说，

如果 A <: B

那么 List[A] <: List[B]

但是这样的协变会导致问题，如下面的java代码

```java

NonEmpty[] a = new NonEmpty[]{new NonEmpty(1, Empty, Empty)}
// 
// 
IntSet[] b = a

b[0] = Empty
NonEmpty s = a[0]
```

最后把一个空值b[0]赋值给了非空的类型`s`，显然是出错了。我们来回顾下整个过程

-  声明一个NonEmpty[] a
- 由于IntSet是NonEmpty的父类型，而且Array是协变的，所以IntSet[]是NonEmpty[]的父类型
- 所以IntSet[] b 可以赋值为NonEmpty[] a

- 当运行到第三行的时候,会抛出一个运行时异常，这是因为java会在声明时将对应的`NonEmpty`类型记录下来，标识整个数组的合法类型
- 当运行时，如果赋值为Empty，检测和之前保存的类型不符，于是抛出异常



这并不是一个好的实践，因为这种类型的错误本该在编译期间就能被检测出来的，这会导致我们付出额外的代价来做运行时的检查。



那么Java的开发者为什么当初没有考虑到这点呢？

其实是赶工的历史原因，按照Martin的说法，早期的Java为了实现排序方法，字符数组和整数数组都能作为参数传入sort()函数内，Array的协变就是必须的，这样才能都以sort(Object[])的方式被传入。但是在Java 5之后，可以通过泛型来实现（所以说这是一个早期设计的遗留问题）



上面的讨论带来一个问题，什么情况下协变是对的，上面情况下协变不应该被使用。

对于这个问题，有一个理论可以参考

- [Liskov_substitution_principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)

非形式化地论述如下：

> 如果 A是B的子类型，那么对B能做的操作，也能对A来做



如果把上面的论断代入具体的例子，非形式化地描述：“人是动物的一种，动物有的特征，人都有”。





```scala
abstract class IntSet 
object Empty extends IntSet 
class NonEmpty(elem: Int, left: IntSet, right: IntSet) extends IntSet 
```

好了，铺垫了这么久，终于到了本小节的习题了

```scala
val a: Array[NonEmpty] = Array(new NonEmpty(1, Empty, Empty))
val b: Array[IntSet] = a
b(0) = Empty
val s: NonEmpty = a(0)
```



当你尝试上述代码时，会发生什么？

- [ ] A type error in line 1
- [ ] A type error in line 2
- [ ] A type error in line 3
- [ ] A type error in line 4
- [ ] Run-time exception
- [ ] No compilation or run-time errors

说实话，我觉得如果不运行代码，这个题目就是初见杀（虽然能排除4个明显错误的选项）



正确答案：在第二行抛出类型错误

因为在Scala中，Array不是协变的

```plain
final class Array[T](_length: Int) extends java.io.Serializable with java.lang.Cloneable
```

也就是说，从 `NonEmpty <: IntSet` 无法推出 `Array[NonEmpty] <: Array[IntSet]`

### *4.4 Varience (Optional)

​	在上一小节我们注意到，`Array`不是协变的，而`List`是协变的。

对于`C[T]`，有`A <: B`

- 如果 `C[A] <: C[B]`  我们说C是协变(covariant)的 可以写作 `class C[+T]`
- 如果`C[A] >: C[B]` 我们说C是逆变(contravariant)的 可以写作 `class C[-T]`
- 如果上面两个条件都不满足 我们说C是不变(nonvariant)的 可以写作 `class C[A]`



那么，对于以下两个类型

```scala
type A = IntSet => NonEmpty
type B = NonEmpty => IntSet
```



A和B的关系如何？

- [ ] A <: B
- [ ] A >: B
- [ ] 没有关系



{% spoiler 答案是 A <: B %}

因为对于`B`能做的操作，我们同样能对`A`来做，反过来就不行。这说明`A`的内涵比`B`多，即`A`是`B`的子类型

> 如果 A是B的子类型，那么对B能做的操作，也能对A来做

这句话有些抽象，当我每次都记不清楚的时候，我都会举例

> 人是动物的一种，动物能行动，人也能行动，人能打字，动物不能打字。
>
> 动物有的属性人都有，人有的属性动物不一定有，人的内涵比动物多，所以人是动物的子类型。

（比喻有时候会偏离事物的原貌，但大多数时候举例+比喻有助于理解,建立一个初步的印象，不准确的部分可以通过后续的学习来补充，如整数和负数的概念引入）

通过上面的例子，我们可以归纳出下面这个规律

如果

- A2 <: A1
- B1 <: B2

那么

A1 => B1 <: A2 => B2

“大到小属于小到大”

通过以上的推导，更进一步地，

T => U <: T- => U+

```scala
trait Function1[-T, +U] {
  def apply(x: T): U
}
```

这样一个函数是协变的。

大多数情况而言，协变类型参数只会出现在函数的结果部分，而逆变类型参数只会出现在函数的参数部分。

 



### 4.5 Decomposition

这一小节主要设计了一个表达式求值的类，通过展示新加的class会以平方的速率（添加两个新的类，函数增加了几十个）增加函数，告诉我们应该合理地设计类的分解关系。



示例代码有些长，这里就不列出来对应的练习了。



### 4.6 Pattern Matching

在scala中，说到pattern matching,就离不开case class

```scala
trait Expr
case class Number(n: Int) extends Expr
case class Sum(e1: Expr, e2: Expr) extends Expr
```

我们首先定义了一个公共的trait,给出它的两个子类。

`case class`定义后会隐式定义它的伴生类和对应的`apply`方法，对于上面的代码，其实隐式地定义了

```scala
object Number {
  def apply(n: Int) = new Number(n)
}

object Sum {
  def apply(e1: Expr, e2: Expr) = new Sum(e1, e2)
}
```



如`Number(4)`展开为`Number.apply(2)`



我们可以用模式匹配来写求值函数

```scala
def eval(e: Expr): Int = e match {
  case Number(n) => n
  case Sum(e1, e2) => eval(e1) + eval(e2)
}
```

看起来语法有些像`switch case`，不过这里可不用`break`

关于`case class` 和`patten matching`的相关知识可以参考《Programming in Scala 3rd Edition》第15章`Case Classes and Pattern Matching`的部分



这里Martin举了个例子来说明模式匹配的原理

对于表达式`eval(Sum(Number(1), Number(2)))`

展开为

```
Sum(Number(1), Number(2)) match {
  case Number(n) => n
  case Sum(e1, e1) => eval(e1) + eval(e2)
}
```

匹配到第二个，化简为

```
eval(Number(1)) + eval(Number(2))
```

到这一步可以显然化简

```
1 + 2
```



这个问题有专门的研究，可以参见[此处](https://www.scala-lang.org/docu/files/TheExpressionProblem.pdf)



现在给出函数签名如下，要求能展示出表达式的字符串

```scala
def show(e: Expr): String = ???
```



下面是一个是实现

```scala
def show(e: Expr): String = e match {
  case Number(x) => x.toString
  case Sum(l, r) => show(l) + " + " + show(r)
}
```



如果要在表达式中添加一个“变量”和“乘法运算”,使得`show`函数能正确的打印出表达式，几个例子如下

```text
Input1:
Sum(Prod(Number(2), Var("x")), Var("y"))

Output1:
2 * x + y

Input2:
Prod(Sum(Number(2), Var("x")), Var("y"))

Output2:
(2 + x) * y
```

注意到尽可能少用括号,这要求我们对乘法加法的运算进行分类讨论，乘法两侧去除多余的括号

下面是一个实现

```scala
trait Expr {
  def eval: Int = this match {
    case Number(n) => n
    case Sum(e1, e2) => e1.eval + e2.eval
    case Prod(e1, e2) => e1.eval * e2.eval
  }

  def show: String = this match {
    case Number(x) => x.toString
    case Sum(l, r) => l.show + " + " + r.show
    case Var(x) => x
    case Prod(Sum(a, b), Sum(c, d)) => 
      s"(${a.show} + ${b.show}) * (${c.show} + ${d.show})"
    case Prod(Sum(a, b), e) => 
      s"(${a.show} + ${b.show}) * ${e.show}"
    case Prod(e, Sum(a, b)) => 
      s"(${e.show} * (${a.show} + ${b.show})"
    case Prod(e1, e2) => 
      s"${e1.show} * ${e2.show}"
  }
}

case class Number(n: Int) extends Expr
case class Sum(e1: Expr, e2: Expr) extends Expr
case class Var(x: String) extends Expr
case class Prod(e1: Expr, e2: Expr) extends Expr
```



验证如下

```scala
scala> Sum(Prod(Number(2), Var("x")), Var("y")).show
res3: String = 2 * x + y

scala> Prod(Sum(Number(2), Var("x")), Var("y")).show
res4: String = (2 + x) * y

```






### 4.7 Lists

对于`List`有一点要注意的，和普通的运算符不一样,`::`是右结合的。

也就是说 `A :: B :: C` 等价于 `A :: (B :: C)`

下面举个展开的例子

```scala
scala> val nums = 1 :: 2 :: 3 :: Nil
nums: List[Int] = List(1, 2, 3)

scala> val numsAnother = Nil.::(3).::(2).::(1)
numsAnother: List[Int] = List(1, 2, 3)

scala> nums == numsAnother
res0: Boolean = true

```

以`:`结尾的运算符/函数 是右结合的。



对于`x :: y :: List(xs, ys) :: zs`, 这个表达式匹配到的长度是多少？



{% spoiler 显然是匹配大于等于3的长度 %}

后面讲了个特别trival的插入排序（为了引出归并排序）的例子，这里就不赘述了。





## 编程作业： Huffman Coding



[Huffman编码](https://en.wikipedia.org/wiki/Huffman_coding)是一种压缩算法。

在普通的未压缩文本中，每个字符由相同的位数表示。在赫夫曼编码中，每个字符都可以具有不同长度的位表示，在文本中经常出现的字符用较短的位序列表示。每个哈夫曼代码都定义了用于表示每个字符的特定位序列。



赫夫曼代码可以用二叉树表示，其叶子表示应该被编码的字符。上面的代码树可以表示A到H的字符。



这个其实大一就应该会讲（不讲的话《数据结构》，《操作系统》总有一个会提到的）

网上的资料也很多，不了解的话，找一篇看下就明白了，这里就不赘述了。

[作业下载地址](http://alaska.epfl.ch/~dockermoocs/progfun1/patmat.zip)

### 构造

假如我们的数据结构实现如下

```scala
abstract class CodeTree

case class Fork(left: CodeTree, right: CodeTree, chars: List[Char], weight: Int) extends CodeTree

case class Leaf(char: Char, weight: Int) extends CodeTree

```



首先，第一部分的任务很简单, 完成`weight`和`chars`的实现

```scala
def weight(tree: CodeTree): Int = ??? // tree match ...
  
def chars(tree: CodeTree): List[Char] = ??? // tree match ...
  
def makeCodeTree(left: CodeTree, right: CodeTree) =
  Fork(left, right, chars(left) ::: chars(right), weight(left) + weight(right))
```

一个实现如下,这里用`_`代替的不需要的变量（减少了起名字的烦恼

```scala
def weight(tree: CodeTree): Int = tree match {
  case Fork(_, _, _, w) => w
  case Leaf(_, w) => w
}
  
def chars(tree: CodeTree): List[Char] = tree match {
  case Fork(_, _, cs, _) => cs
  case Leaf(c, _) => List(c)
}
```



接下来，我们需要实现从字符串编码到Huffman树的部分。

部分完成的函数和函数签名如下

```scala
def string2Chars(str: String): List[Char] = str.toList

// times(List('a', 'b', 'a')) =>  List(('a', 2), ('b', 1))
def times(chars: List[Char]): List[(Char, Int)] = ???

// 返回一个List[Leaf],内部按照Left的weight从小到大排序
//  List(('a', 2), ('b', 1)) =>  List(Leaf('b', 2), Leaf('a', 1))
def makeOrderedLeafList(freqs: List[(Char, Int)]): List[Leaf] = ???

// 判断Huffman树是否只有一个节点
def singleton(trees: List[CodeTree]): Boolean 
```

这部分比较容易想出来，`times`函数的实现，如果你看过spark的word-count例子，就会发现这个简直一摸一样，实现方法多样，个人惯用的方法是

```scala
def times(chars: List[Char]): List[(Char, Int)] = 
  chars.groupBy(identity).toList.map(x => (x._1, x._2.length))
```

下面构造有序叶子List也有多种实现，个人习惯用sortBy来解决这种不算特别复杂，需要多个维度排序的问题

```scala
def makeOrderedLeafList(freqs: List[(Char, Int)]): List[Leaf] = 
  freq.sortBy(x => (x._2, x._1))(Ordering.Tuple2(Ordering.Int, Ordering.Char)).map(x => Leaf(x._1, x._2))
```

这里的排序是先按照`weight`升序，再按照`char`升序，更加复杂的排序逻辑可以用`sortWith`实现

判断是否只有一个节点是一个送分题

```scala
def singleton(trees: List[CodeTree]): Boolean = 
  trees.length == 1
```



下面几个函数是编码方面的难点

```scala
def combine(trees: List[CodeTree]): List[CodeTree] = ???

def until(xxx: ???, yyy: ???)(zzz: ???): ??? = ???

def createCodeTree(chars: List[Char]): CodeTree = ???
```

`combine`函数做的事，其实就是编码是从下到上的合并操作。传入的`trees`是有序的节点，每次从里面取出两个进行合并，然后插入回去，维护数组的有序状态。当数组中的节点数小于2时停止。

当然可以`if(trees.length >= 2) ...`这样写，但是更好的做法是使用模式匹配

```scala
def combine(trees: List[CodeTree]): List[CodeTree] = trees match {
  case left :: right :: cs => 
    (makeCodeTree(left, right) :: cs).sortWith((t1, t2) => weight(t1) < weight(t2))
  case _ => trees
}
```

从这里开始，我们会经常复用写好的代码了。`case left :: right :: cs` 匹配的是一个长度大于等于2的`List[CodeTree]`。（还记得4-7的练习吗？）匹配完成后，我们取到权重最小的这两个节点（因为`trees`本身就排好序了），将它俩合并成一个`Fork`节点，在放回去。

为啥我取名叫 `left` `right` `cs` , 因为`cs`是`codeTrees`的缩写，`left`和`right`很有树的风格<del>好吧，我承认自己不会取名</del> 



这里直接放回去并不能保证有序，所以后面还加了一个`sortWith`,按照节点的`weight`来排序。

对于其他的情况（节点数小于2）， 我们直接返回树它本身。



下面`until`的实现就比较麻烦了，因为给出的函数签名不完整`def until(xxx: ???, yyy: ???)(zzz: ???): ??? = ???`。这么多问号闹哪样嘛。

还好，注释里详细描述了函数的调用方式`until(singleton, combine)(trees)`根据这个调用方式，我们可以依据类型补全部分函数签名为

```scala
def until[T](xxx: List[T] => Boolean, yyy: List[T] => List[T])(zzz: List[T]): List[T] 
```

其实这里完全可以不用使用泛型，用`CodeTree`就行了。但我想到这其实能用来表达一个通用的控制流程，逻辑大致为，输入数据`zzz`，当结果不满足`xxx(zzz)`时，递归调用`until(xxx, yyy)(yyy(zzz))`

都是xyz看的头大？我们换一个具体的例子(面向函数签名编程)

```scala
def until(stop: List[Int] => Boolean, 
          action: List[Int] => List[Int])
         (data: List[Int]): List[Int] = {
           if (stop(data)) data
           else until(stop, action)(action(data))
         }

scala> until((x: List[Int]) => x.head < 12, (xs: List[Int]) => (xs.head - 1) :: xs)(List(24))
res8: List[Int] = List(11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24)
```

换上名字在加上例子，是不是容易理解多了？这个例子中`until`传入两个匿名函数`(x: List[Int]) => x.head < 12`, `(xs: List[Int]) => (xs.head - 1) :: xs`, 数据的初始状态是`List(24)`,当数组中的第一个元素大于等于12的时候，会在数组前面添加一个新的比原来小1的元素，构成一个新的数组。



回到我们原来的问题，结果就很自然了

```scala
def until[T](stop: List[T] => Boolean, action: List[Int] => List[Int])(data: List[T]): List[T] = {
  if (stop(data)) data
  else until(stop, action)(action(data))
}
```

其实这种写法在之前的课堂练习上有讲过。

现在准备工作都基本完成了，接下来是Huffman树的编码部分

```scala
def createCodeTree(chars: List[Char]): CodeTree = ???
```

我们需要把一个字符数组映射成Huffman CodeTree.

显然需要复用之前的结果，看看函数签名就好了，哪个函数是以`List[Char]`作为传入参数的，嗯，找到`times`,

`times(chars)`返回的类型是`List[(Char, Int)]`,正好`makeOrderedLeafList`需要这个类型的参数,返回的结果是`List[Leaf]`，这时候我们就能用`until(singleton, combine)(trees)`来构造Huffman CodeTree了。

大功告成，回过头来看是不是有一种水到渠成的感觉？

``` scala
def createCodeTree(chars: List[Char]): CodeTree = {
  val nodes = makeOrderedLeafList(times(chars))
  until(singleton, combine)(nodes).head
}
```



### 解码

上面我们实现了从`List[Char]`到`CodeTree`的构造过程，接下来我们要完成编码过程

给定Huffman树的结构和码表，逆推出原始字符数组

```scala
type Bit = Int

def decode(tree: CodeTree, bits: List[Bit]): List[Char] = ???

// 这个是法语中各个字母的出现次数，现在如果我们想对一段法语进行压缩，就要先构造出它的Huffman Tree，这里已经给出，数据来源：http://fr.wikipedia.org/wiki/Fr%C3%A9quence_d%27apparition_des_lettres_en_fran%C3%A7ais
// PS, 但是看起来数据有些对不上，应该是做过进一步的处理的
val frenchCode: CodeTree = Fork(Fork(Fork(Leaf('s',121895),Fork(Leaf('d',56269),Fork(Fork(Fork(Leaf('x',5928),Leaf('j',8351),List('x','j'),14279),Leaf('f',16351),List('x','j','f'),30630),Fork(Fork(Fork(Fork(Leaf('z',2093),Fork(Leaf('k',745),Leaf('w',1747),List('k','w'),2492),List('z','k','w'),4585),Leaf('y',4725),List('z','k','w','y'),9310),Leaf('h',11298),List('z','k','w','y','h'),20608),Leaf('q',20889),List('z','k','w','y','h','q'),41497),List('x','j','f','z','k','w','y','h','q'),72127),List('d','x','j','f','z','k','w','y','h','q'),128396),List('s','d','x','j','f','z','k','w','y','h','q'),250291),Fork(Fork(Leaf('o',82762),Leaf('l',83668),List('o','l'),166430),Fork(Fork(Leaf('m',45521),Leaf('p',46335),List('m','p'),91856),Leaf('u',96785),List('m','p','u'),188641),List('o','l','m','p','u'),355071),List('s','d','x','j','f','z','k','w','y','h','q','o','l','m','p','u'),605362),Fork(Fork(Fork(Leaf('r',100500),Fork(Leaf('c',50003),Fork(Leaf('v',24975),Fork(Leaf('g',13288),Leaf('b',13822),List('g','b'),27110),List('v','g','b'),52085),List('c','v','g','b'),102088),List('r','c','v','g','b'),202588),Fork(Leaf('n',108812),Leaf('t',111103),List('n','t'),219915),List('r','c','v','g','b','n','t'),422503),Fork(Leaf('e',225947),Fork(Leaf('i',115465),Leaf('a',117110),List('i','a'),232575),List('e','i','a'),458522),List('r','c','v','g','b','n','t','e','i','a'),881025),List('s','d','x','j','f','z','k','w','y','h','q','o','l','m','p','u','r','c','v','g','b','n','t','e','i','a'),1486387)


// 下面这段消息是经过Huffman 编码后的数据
// 原文是什么呢？
val secret: List[Bit] = List(0,0,1,1,1,0,1,0,1,1,1,0,0,1,1,0,1,0,0,1,1,0,1,0,1,1,0,0,1,1,1,1,1,0,1,0,1,1,0,0,0,0,1,0,1,1,1,0,0,1,0,0,1,0,0,0,1,0,0,0,1,0,1)
```

需要实现的函数其实只有一个，但是却需要一定的思考

```scala
def decode(tree: CodeTree, bits: List[Bit]): List[Char] = {
  def traverse(remaining: CodeTree, bits: List[Bit]): List[Char] = remaining match {
    case Leaf(c, _) if bits.isEmpty => List(c)
    case Leaf(c, _) => c :: traverse(tree, bits)
    case Fork(left, _, _, _) if bits.head == 0 => traverse(left, bits.tail)
    case Fork(_, right, _, _) => traverse(right, bits.tail)
  }
  traverse(tree, bits)
}
```

Huffman编码生成的序列有一个规则，就是在码表中的任意一个编码，都不会是其他任意一个编码的前缀，这种叫前缀码。

因为前缀码的性质，带来一个好处，一旦匹配到合适的编码，立刻能解码而不用担心重复的问题。

上述代码的思路如下

- 如果匹配到叶子节点，并且剩余的编码解析完毕，返回结果
- 如果匹配到叶子节点，剩余的编码尚未解析完毕，将当前结果和递归调用的剩余编码的解码结果进行合并
- 如果匹配到非叶子节点，那么左边的如果当前码表为0，在左子树匹配
- 否则匹配右子树



上面问题的答案通过`sbt console`可以方便的得到

```scala
sbt console 
scala> import patmat.Huffman

scala> Huffman.decodedSecret.mkString
res3: String = huffmanestcool
```

这句话的句读为`huffman est cool`(类似哈夫曼赛高的意思，嗯，好冷)

### 编码

有解码自然就有编码。

```scala
def encode(tree: CodeTree)(text: List[Char]): List[Bit] = ???
```

给出Huffman CodeTree的结构和字符串数组，求编码结果

```scala
def encode(tree: CodeTree)(text: List[Char]): List[Bit] = {
  def find(tree: CodeTree)(c: Char): List[Bit] = tree match {
    case Leaf(_, _) => Nil
    case Fork(left, _, _, _) if chars(left).contains(c) => 0 :: find(left)(c)
    case Fork(_, right, _, _) if chars(right).contains(c) => 1 :: find(right)(c)
  }
  text.flatMap(find(tree))
}
```

有了上面的经验，可以很快的仿照写出这样的实现。

这里用了一个小技巧，将`find`函数`curry`化，可以使得`flatMap`里面少写一个参数。

```scala
type CodeTable = List[(Char, List[Bit])]

def codeBits(table: CodeTable)(char: Char): List[Bit] = ???

def convert(tree: CodeTree): CodeTable = ???

def mergeCodeTables(a: CodeTable, b: CodeTable): CodeTable = ???

def quickEncode(tree: CodeTree)(text: List[Char]): List[Bit] = ???
```

上面的编码实现很自然，但是查找效率太低了，因为对于每一个字符都需要去遍历整个树，其实我们可以对所有字符集做一次计算，将结果保存，之后直接查表就行了。

现在需要用以上的辅助函数来实现一个更加高效的版本。

```scala
// 返回查表结果
def codeBits(table: CodeTable)(char: Char): List[Bit] = 
  table.filter(code => code._1 == char).head._2

// 将树结构转换成表
def convert(tree: CodeTree): CodeTable = tree match {
  case Leaf(c, _) => List((c, Nil))
  case Fork(left, right, _, _) => mergeCodeTables(convert(left), convert(right))
}

// 合并两个码表
def mergeCodeTables(a: CodeTable, b: CodeTable): CodeTable = {
  a.map(code => (code._1, 0::code._2)) ++ b.map(code => (code._1, 1::code._2))
}

def quickEncode(tree: CodeTree)(text: List[Char]): List[Bit] =
  text.flatMap(codeBits(convert(tree)))
```



至此，作业代码完成，完整代码可参考[此处](https://gist.github.com/counter2015/d80fbf1442a70a3f764d0614c6b4feb5)