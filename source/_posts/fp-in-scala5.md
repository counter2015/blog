# Progfun 5. Lists

title: Progfun 5. Lists
date: 2020-03-09 22:06:38
tags: [scala, 读书笔记, Coursera] 
description:  这是一篇拖更了6个月的文章。
categories: [技术]

------



### 5.1 More Functions on Lists

实现`List`的`init`函数，它会返回除了最后一个元素的`List`中的其他所有元素

```scala
def init[T](xs: List[T]): List[T] = xs match {
  case List() => throw new Error("init of empty list")
  case List(x) => List()
  case y :: ys => y :: init(ys)
}
```

实现`List`的`remove`函数

`removeAt(1, List('a', 'b', 'c', 'd')) // List(a, c, d)`

```scala
def removeAt[T](n: Int, xs: List[T]): List[T] = xs.take(n) ++ xs.drop(n+1)
```

实现`flatten`函数

`flatten(List(List(1, 1), 2, List(3, List(5, 8)))) // List(1, 1, 2, 3, 5, 8)`

```scala
def flatten(xs: List[Any]): List[Any] = xs match {
  case List() => List()
  case (head: List[_]) :: tail => flatten(head) ++ flatten(tail)
  case head :: tail => head :: flatten(tail)
}
```



### 5.2 Pairs and Tuples

马丁又在tuple里玩(answer,42)的梗

<del>冷幽默梗都来自程序员</del>

首先给展示了一个fp(function programming)的merge sort(归并排序)实现

```scala
def msort(xs: List[Int]): List[Int] = {
  val n = xs.length / 2
  if (n == 0) xs
  else {
    def merge(xs: List[Int], ys: List[Int]) = ???
    val (first, second) = xs splitAt n 
    merge(msort(first), msort(second))
  }
}


def merge(xs: List[Int], ys: List[Int]): List[Int] = 
  xs match {
    case Nil => ys
    case x :: xs1 =>
      ys match {
        case Nil => xs
        case y :: ys1 =>
          if (x < y) x :: merge(xs1, ys)
          else y :: merge(xs, ys1)
      }
  }
```

这里主要关注的是归并部分的实现，大致的思路不难理解，对于两个List，如果有一个是空的，就返回另外一个

否则，如果取出各自的第一个元素进行比较，将小的那个放在前面，后面的部分递归的再次执行归并的操作



这样写有一个问题，就是不美观。

在我上面的描述中，明明应该是对称的，但是代码却没体现出来，还写得很难阅读

于是马丁给我们留了习题，用下面的形式结合Tuple写个更好看的版本出来。

```scala
def merge(xs: List[Int], ys: List[Int]): List[Int] = 
	(xs, ys) match {
    case (Nil, ys) => ys
    case (xs, Nil) => xs
    case (x :: xs1, y :: ys1) => 
      if (x < y) x :: merge(xs1, ys)
      else y :: merge(xs, ys1)
  }
```

有的人可能会奇怪，两个都为空的情况不用判断了？

其实，这种情况已经包含在 `case (Nil, ys)` 里了，不要因为变量是`ys`就相当然得认为不能为`Nil`

这里其实等价于 `if (xs.length == 0) ...`

那么这个`ys`是否和外部merge函数传入的`ys`的值完全一致呢？

在这个例子里是的，但是换个例子则未必

如

```scala
scala> def f(xs: List[Int], ys: List[Int]): List[Int] = 
     |   (xs, ys) match {
     |     case (ys, _) => ys
     |     case _ => List(-1)
     |   }
f: (xs: List[Int], ys: List[Int])List[Int]
```

假如我们定义上述函数`f`，想判断当`xs` 和`ys`相同时返回`ys`,但是结果呢

```scala
scala> f(List(1, 2), List(2, 3))
res6: List[Int] = List(1, 2)
```

这是由于内部的`ys`和外部的并不相同，只是凑巧起了个名字

<del>一个叫小明的人和一个叫小明的狗显然不是同样的东西</del>

它其实指代的时Tuple中匹配到的第一个元素，然后 命名为ys。

如果想用正确的方式匹配和外部`ys`变量一样的值，有办法吗？

```scala
scala> def f(xs: List[Int], ys: List[Int]): List[Int] = 
     |   (xs, ys) match {
     |     case (`ys`, _) => `ys`
     |     c
     |   }
f: (xs: List[Int], ys: List[Int])List[Int]

scala> f(List(1, 2), List(2, 3))
res7: List[Int] = List(-1)

scala> f(List(1, 2), List(1, 2))
res8: List[Int] = List(1, 2)
```

用反引号就行了



### 5.3 Implicit Parameters

上一小节我们简单实现了整数数组的归并排序，有没有办法把之前的方案稍微做一些改造，

使得能够适用于多种不同类型的情况呢？

```scala
def msort[T](xs: List[T])(lt: (T, T) => Boolean): List[T] = {
  val n = xs.length / 2
  if (n == 0) xs
  else {
    def merge(xs: List[T], ys: List[T]): List[T] = (xs, ys) match {
      case (Nil, ys) => ys
      case (xs, Nil) => xs
      case (x :: xs1, y :: ys1) =>
        if (lt(x, y)) x :: merge(xs1, ys)
        else y :: merge(xs, ys1)
    }
    val (first, second) = xs splitAt n 
    merge(msort(first)(lt), msort(second)(lt))
  }
}
```

在REPL里进行一些简单的测试

```scala
scala> val nums = List(2, 3, -2, 5, 1)
nums: List[Int] = List(2, 3, -2, 5, 1)

scala> msort(nums)((x: Int, y: Int) => x < y)
res4: List[Int] = List(-2, 1, 2, 3, 5)

scala> val fruits = List("banana", "apple", "peach", "watermelon", "orange")
fruits: List[String] = List(banana, apple, peach, watermelon, orange)

scala> msort(fruits)((x: String, y: String) => x.compareTo(y) < 0)
res7: List[String] = List(apple, banana, orange, peach, watermelon)
```

这里是传入了一个自定了的`lt`函数来确定元素之间的大小关系，其实，标准库里就有类似的功能。

```scala
import scala.math.Ordering
def msort[T](xs: List[T])(implicit ord: Ordering[T]): List[T] = {
  val n = xs.length / 2
  if (n == 0) xs
  else {
    def merge(xs: List[T], ys: List[T]): List[T] = (xs, ys) match {
      case (Nil, ys) => ys
      case (xs, Nil) => xs
      case (x :: xs1, y :: ys1) =>
        if (ord.lt(x, y)) x :: merge(xs1, ys)
        else y :: merge(xs, ys1)
    }
    val (first, second) = xs splitAt n 
    merge(msort(first)(ord), msort(second)(ord))
  }
}

scala> msort(nums)(Ordering.Int)
res8: List[Int] = List(-2, 1, 2, 3, 5)

scala> msort(nums)(Ordering.Int.reverse)
res10: List[Int] = List(5, 3, 2, 1, -2)
```

有的时候我们希望能按默认顺序排序，想省略掉后面柯里化的第二个关于顺序的传入参数。

这时候就可以用`implicit`,函数签名做如下修改

`def msort[T](xs: List[T])(implicit ord: Ordering[T]): List[T]`

这样依赖，当我们传入的数组类型为`List[Int]`时，编译器会帮助做类型推断，使用Ordering[Int]自动补齐第二个入参

```scala
scala> msort(nums)
res11: List[Int] = List(-2, 1, 2, 3, 5)
```

### 5.4 Higher-Order List Functions

​	下面是模式匹配和高阶函数来实现两种将`List`中每个元素平方的函数签名（尚未实现）

```scala
def squareList(xs: List[Int]): List[Int] =
  xs match {
    case Nil => ???
    case y :: ys => ???
  }

def squareList(xs: List[Int]): List[Int] =
  xs map ???
```

完成上面这两个函数很简单

```scala
def squareList(xs: List[Int]): List[Int] =
  xs match {
    case Nil => Nil
    case y :: ys => (y * y) :: squareList(ys)
  }

def squareList(xs: List[Int]): List[Int] =
  xs map (x => x * x)
```

可以看出，下面这个写法显然更简洁易懂。



如果我们要把`List`中的元素分组

```scala
pack(List("a", "a", "a", "b", "c", "c", "a")) shouldBe 
List(
  List("a", "a", "a"),
  List("b"),
  List("c", "c"),
  List("a")
    )
```

对应函数的模板如下

```scala
def pack[T](xs: List[T]): List[List[T]] = xs match {
  case Nil => Nil
  case x :: xs1 =>
    ???
}
```

实现如下

```scala
def pack[T](xs: List[T]): List[List[T]] = xs match {
  case Nil => Nil
  case x :: xs1 =>
    val (a, b) = xs.span(_ == x)
    a :: pack(b)
}
```

在已经实现`pack`函数的基础上，我们希望对每个元素进行计数

```scala
encode(List("a", "a", "a", "b", "c", "c", "a")) shouldBe
List(("a", 3), ("b", 1), ("c", 2), ("a", 1))
```

这一看就是稍微变化下就好了

```scala
def encode[T](xs: List[T]): List[(T, Int)] =
	pack(xs).map(ys => (ys.head, ys.length))
```



### 5.5 Reduction of Lists

`reduceLeft`和`reduceRight`有时候得到的结果类似，但是效率不同。但在有的情况下，只有一种是适合的。

```scala
def concat[T](xs: List[T], ys: List[T]): List[T] = 
  (xs foldRight ys) (_ :: _)
```

比如这种情况下，只有`foldRight`是合适的

为什么呢

- [x]  The types would not work out
- [ ]  The resulting function would not terminate
- [ ]  The result would be reserved

事实上会报这个错误

`error: value :: is not a member of type parameter T`

这是由于

```scala
def foldLeft[B](z: B)(op: (B, T) => B)
```

那么，上面的代码修改成`foldLeft后`

```scala
(xs foldLeft ys) (_ :: _)
=> xs.foldLeft(ys)((x: List[T], y: T) => x :: y)
```

显然 `List[T] :: T`是不可能通过编译的，因为`::`函数是右结合的，而只有`List[T]`有定义`::`方法。



在上面的基础下，我们来看这部分代码

```scala
def mapFun[T, U](xs: List[T], f: T => U): List[U] =
  (xs foldRight List[U]())( ??? )

def lengthFun[T](xs: List[T]): Int =
  (xs foldRight 0)( ??? )
```

可以看出想让我们完成`map`和`length`方法，一个是对`List`内的元素做变换，一个是求`List`的长度

```scala
def mapFun[T, U](xs: List[T], f: T => U): List[U] = 
  (xs foldRight List[U]())((a: T, b: List[U]) => f(a) :: b)

def lengthFun[T](xs: List[T]): Int =
  (xs foldRight 0)((a: T, b: Int) => b + 1)
```

在REPL中的测试结果如下

```scala
scala> lengthFun(List(1,23,4))
res2: Int = 3

scala> mapFun(List(2,3,2), (a: Int) => a + 1)
res3: List[Int] = List(3, 4, 3)
```

按照函数签名来填空，一切都水到渠成。

### 5.6 Reasoning About Concat

再次回顾之前的连接运算符`++`

```
(xs ++ yz) ++ zs = xs ++ (ys ++ zs)
xs ++ Nil = xs
Nil ++ xs = xs
```

我们可以看到，这个运算满足交换律，结合律，存在幺元。<del>交换幺半裙</del>

如果我们想要证明这种性质，可以使用数学归纳法

> 为了证明P(n) 对于所有的 n >= b都成立，首先需要证明
>
> 1. 满足P(b)
> 2. 假设P(n)成立， 对于所有n >= b的情况，如果P(n)成立，那么有P(n+1)成立

听起来有点拗口，但是这个应该在高中数学课堂上就曾经讲过。

这里不再赘述其正确性和例子了，<del>忘记的可以回去翻翻课本</del>

上面的理论之所以能对纯函数式语言起作用，是因为不会产生副作用，将一段代买简化规约后的结果和它原始的结果相同。

这个原则也被称之为**引用透明**（referential transparency）

同样的套路

- P（`Nil`）满足条件
- 夹着P(`xs`)成立, 如果`xs`有不少于0个元素，对于某个元素`x`, 且P(`xs`)能推导出P（`x::xs`）

让我们来试着证明下面的式子

`(xs ++ ys) ++ zs = xs ++ (ys ++ zs)`

```scala
def concat[T](xs: List[T], ys: List[T]): List[T] = xs match {
  case Nil => ys
  case x :: xs1 => x :: concat(xs1, ys)
}
```

对于P（`Nil`） = 

`(Nil ++ ys) ++ zs = ys ++ zs == Nil ++ (ys ++ zs)`

显然成立

而对于P(`x :: xs`) =

左边 = ((x :: xs) ++ ys) ++ zs

= (x :: (xs ++ ys)) ++ zs // 根据代码`case x :: xs1 => x :: concat(xs1, ys)`

= x :: ((xs ++ ys) ++ zs) // 根据代码`case x :: xs1 => x :: concat(xs1, ys)`

= x :: (xs ++ (ys ++ zs)) // 由假设P(`xs`)成立可得

右边 = (x :: xs ) ++ (ys ++ zs)

= x :: (xs ++ (ys ++ zs))

左边 = 右边

由数学归纳法可知，原命题成立。



>  有人说：Haskell科技树是
>
> 文法 --> lambda运算 --> 简单类型和函数类型 --> 代数数据类型 --> 类型类，高阶类型和范畴 --> 面向组合子编程 --> 编译器魔法，DSL和协议建模等等 --> 程序正确性证明
>
> Scala科技树是
>
> 面向对象 --> 高阶函数 --> 简单的代数数据类型+模式匹配 --> 函数类型，子类型及协变/逆变（Scala对象和类型系统的重点） --> 接口和泛型，更优雅的设计模式 --> 命名空间和模块设计 --> 分布式和高并发场景的业务实现
>
> 那么我们到达了编程的最高境界么（大雾

 

### 5.7 A Larger Equational Proof on Lists

这一章主要讲的是如何在函数式编程语言中使用形式化证明

假设我们已知如下两个式子

```scala
Nil.reverse = Nil // 式1 来自于不证自明的事实
(x :: xs).reverse = xs.reverse ++ List(x) // 式2 来源于reverse函数的定义
```

我们需要证明`xs.reverse.reverse = xs`



记得上一小节的证明方法吗，我们需要使用数学归纳法，首先要找到满足的初始条件

命题`P(xs): xs.reverse.reverse = xs`

显然，对于`Nil`时，命题成立

```scala
P(Nil) = Nil.reverse.reverse // 来自于原命题
= Nil.reverse // 依据式1和链式函数的计算法则
= Nil // 依据式1
```

假设命题对于`P(xs)`时成立

我们有`P(xs): xs.revser.reverse = xs // 式3 基于数据归纳法的假设`


```scala
左边 = P(x :: xs) = (x :: xs).reverse.reverse  // 命题展开
= (xs.reverse ++ List(x)).reverse // 基于式2展开


右边 = x :: xs
= x :: xs.reverse.reverse // 根据归纳假设
```

到了这一步，发现好像两边还是不能凑成一样的形式，这时候就需要仔细观察了

```scala
令 ys = xs.reverse
左边 = (ys ++ List(x)).reverse
右边 = (x :: ys.reverse)
```

要证明原命题`P(xs)`成立，只需要证明`Q(x, ys): (ys ++ List(x)).reverse = (x :: ys.reverse)`

对于命题Q的初始条件，带入`ys = Nil`

```scala
Q(x, Nil) = (Nil ++ List(x)).reverse // 代入展开
= List(x).reverse // 根据 Nil 与 ++ 的计算规律
= (x :: Nil).reverse // 显然 List(x) = x :: Nil
```

左边=右边，初始条件成立

对于Q命题的数学归纳法，假设有

`(ys ++ List(x)).reverse = (x :: ys.reverse) // 式4 根据数学归纳法成立`

那么

```scala
左边 = Q(x, y :: ys) = ((y :: ys) ++ List(x)).reverse // 代入展开
= (y :: (ys ++ List(x))).reverse // 根据List运算的结合律
= (ys ++ List(x)).reverse ++ List(y) // 根据式2变换
= x :: ys.reverse ++ List(y) // 根据式4归纳假设
= x :: (ys.reverse ++ List(y)) // 根据List运算的结合律
= x :: (y :: ys).reverse // 根据式2变换

右边 = x :: (y :: ys).reverse 
```

左边=右边，命题Q成立，由于命题Q成立对应的等价命题P成立

Q.E.D

---

这小节的练习是证明map的运算规律

命题R： 对于任意的List`xs, ys`,函数`f`

已知： 
`Nil map f = Nil // 式1`
`(x :: xs) map f = f(x) :: xs map f // 式2`

证明， `(xs ++ ys) map f = (xs map f) ++ (ys map f)`

我们先来审视一下这个命题，里面有多达三个变量`xs, ys 和 f`

然而证明过程中我们不需要关心所有的变量

考虑到`++`和`map`函数都是左结合的，其实我们需要证明的是

**命题R(xs): 对于任意的xs, ys, f， (xs ++ ys) map f = (xs map f) ++ (ys map f)**

对于初始条件命题`R(Nil)`

```scala
左边 = (Nil ++ ys) map f
= ys map f // 根据 ++ 的运算规律

右边 = (Nil map f) ++ (ys map f)
= Nil ++ (ys map f) // 根据式1
= ys map f // 根据 ++ 的运算规律
```

左边等于右边，所以`R(Nil)`成立

下面假设`R(xs)`成立，需要证明`R(x :: xs)`也成立

```scala
左边 = ((x :: xs) ++ ys) map f
= (x :: (xs ++ ys)) map f // 根据列表连接的性质
= f(x) :: ((xs ++ ys) map f) // 根据式2

右边 = ((x :: xs) map f) ++ (ys map f) 
= (f(x) :: (xs map f)) ++ (ys map f) // 根据式2
= f(x) :: ((xs map f) ++ (ys map f)) // 根据列表连接的性质
= f(x) :: ((xs ++ ys) map f) // 根据归纳假设
```

左边 = 右边

Q.E.D