# Progfun 2.High Order Fuctions

title: Progfun 2.High Order Fuctionst
date: 2019-08-03 22:32:30
tags: [scala, 读书笔记, Coursera] 
categories: [技术]
mathjax: true

------



## 2.1 High Order Fuctions



使用递归的方式计算






$$
\sum(f)(a,b) = \sum_{i=a}^{i <= b}f(i)
$$



```scala
def sum(f: Int => Int)(a: Int, b: Int): Int = {
  def loop(a: Int, acc: Int): Int = {
    if (a > b) acc
    else loop(a + 1, f(a) + acc)
  }
  loop(a, 0)
}

scala> sum(x => x * x)(1, 5)
res0: Int = 55
```



## 2.2 Currying

其实一不小心，上面那个函数的写法就用到了`Currying`，中文名称叫柯里化。

[Haskell Brook Curry](<https://en.wikipedia.org/wiki/Haskell_Curry>)， 数学家，逻辑学家，名字里的每一段都被实现成了编程语言，其中最有名的就是Haskell。

所谓柯里化，就是把一个接受多个参数的函数，可以等价成依次传入多个参数的的函数。

如f(a,b,c) = f(a)(b)(c)

```scala
scala> def sum(a: Int)(b: Int): Int = a + b 
sum: (a: Int)(b: Int)Int

scala> val add1 = sum(1) _
add1: Int => Int = <function1>

scala> add1(22)
res3: Int = 23
```

这里`sum(1) _` 中的`_`指的是省略的参数。

计算
$$
\prod_{i=a}^{i<=b}f(i)
$$

```scala
def product(f: Int => Int)(a: Int, b: Int): Int = {
  if (a > b) 1 
  else f(a) * product(f)(a + 1, b)
}

scala> product(x => x * x)(3, 4)
res4: Int = 144

```

计算阶乘

``` scala
def fact(n: Int): Int = product(x => x)(1, n)

scala> fact(4)
res5: Int = 24
```

下面是一个较为复杂的例子, 把上面的几个函数做了个通用的抽象

```scala
def mapReduce(f: Int => Int, combine: (Int, Int) => Int, zero: Int)(low: Int, high: Int): Int = {
  if (low > high) zero
  else combine(f(low), mapReduce(f, combine, zero)(low + 1, high))
}
  

def product(f: Int => Int)(a: Int, b: Int): Int = mapReduce(f, (x, y) => x * y, 1)(a, b)

def sum(f: Int => Int)(a: Int, b: Int): Int = mapReduce(f, (x, y) => x + y, 0)(a, b)

scala> product(x => x * x)(3, 4)
res6: Int = 144

scala> sum(x => x * x)(1, 5)
res7: Int = 55

```

## 2.3 Example: Finding Fixed Points

我们把

`{x | f(x) = x}`的点称之为`fix point`

这一节使用牛顿迭代法来演示，之前在第一周的笔记中已经实现了，这里就不再赘述。

## 2.4 Scala Syntax Summary

语法类型讲解，从略， 2.5没有习题，从略。

## 2.6 More Fun With Rations

这里讲了个求最大公约数时，可能遇到的整数溢出问题，这里的选项第一开始我还以为没有问题。。。

## 2.7 Evaluation and Operators

这一小节讲的是计算符号的顺序,计算符的顺序除了赋值操作符外，

都是按照**首个字母**来判断优先级，`*+`的优先级等于`**`， 但是`*+`的优先级高于`+*`（下面是一个例子

```
(所有赋值操作符 如，=, *=, +=, /=)
(all letters)
|
^
&
< >
= !
:
+ -
* / %
(all other specail characters)
```

上表中，从上往下，计算的优先顺序依次增加, 同一行的计算符有相同的优先级，相同的优先级情况下从左往右依次计算。

```scala
class X(val value: String) {
  def *+(that: X): X = {
    val newValue = "(" + this.value + "*+" + that.value + ")"
  	new X(newValue)
  }
  
  def **(that: X): X = {
    val newValue = "(" + this.value + "**" + that.value + ")"
    new X(newValue)
  }
  
  def +*(that: X): X = {
    val newValue = "(" + this.value + "+*" + that.value + ")"
    new X(newValue)
  }
  
  override def toString: String = this.value
}

scala> val a = new X("2")
a: X = 2

scala> val b = new X("3")
b: X = 3

// *+ 和 ** 的优先级相同
scala> a *+ b ** a
res11: X = ((2*+3)**2)

scala> a ** b *+ a
res12: X = ((2**3)*+2)

// *+ 的优先级高于 **
scala> a *+ b +* a
res13: X = ((2*+3)+*2)

scala> a +* b *+ a
res14: X = (2+*(3*+2))

```





那么，下述表达式的计算顺序如何？

`a + b ^? c ?^ d less a ==> b | c`

其中 a, b, c这三者代表变量，其他的字符都是运算符

对照上表，依次加上括号

```
  a + b ^? c ?^ d less a ==> b | c
= a + b ^? (c ?^ d) less a ==> b | c
= (a+b) ^? (c ?^ d) less a ==> b | c
= (a + b) ^? (c ?^ d) less (a ==> b) | c
= ((a + b) ^? (c ?^ d)) less (a ==> b) | c
= ((a + b) ^? (c ?^ d)) less ((a ==> b) | c)
```



>  虽然说我们可以查表来计算运算符的优先级，但是更好的办法是直接标注括号，
这样可能你还会减少别人用操作符表示法对你表示不满的频率
比如大声地说这是"bills !*&^%~ code!"

<del>上述代码会被Scala编译器"翻译成" (biills.!*&^%~(code)).!() "</del>

如果你不是为了可以出题为难别人，那么，为了减少挨打的概率，我建议你还是多写几个括号比较好



## 编程作业： Fucntional Sets

这里尝试实现函数式的`Set`



询问一个整数，如果数字在`Set`中返回`true`，不存在返回`false`

首先我们来实现简单的只含一个元素的`Set`

```scala
package funsets 

object FunSets {
  type Set = Int => Boolean
  
  def contains(s: Set: elem: Int): Boolean = s(elem)
  
  def singletonSet(elem: Int): Set = ???
  
  /**
   * Displays the contents of a set
   */
  def toString(s: Set): String = {
    val xs = for (i <- -bound to bound if contains(s, i)) yield i
    xs.mkString("{", ",", "}")
  }

  /**
   * Prints the contents of a set on the console.
   */
  def printSet(s: Set) {
    println(toString(s))
  }
}
```

这个和我们平时接触的`Set`不一样，它存储的是映射关系，而不是具体的数据，所以不会存到类似数组，列表这样的结构里

那么`singletonSet`构造时应该对于所有不等于`elem`的查询返回`false`，对于等于`elem`的查询返回true，里面存放的是这样一个函数

```scala
 /**
   * Returns the set of the one given element.
   */
def sinlgetonSet(elem: Int): Set = x: Int => (x == elem)
```

接下来我们需要实现集合的∩/∪/差集 

求S和T的差集： S - T = {x | x ∈ S & x ∉ T}

``` scala
/**
   * Returns the union of the two given sets,
   * the sets of all elements that are in either `s` or `t`.
   */
def union(s: Set, t: Set): Set = x: Int => contains(s, x) || contains(t, x)

/**
   * Returns the intersection of the two given sets,
   * the set of all elements that are both in `s` and `t`.
   */
def intersect(s: Set, t: Set): Set = (x:Int) => contains(s, x) && contains(t, x)

/**
   * Returns the difference of the two given sets,
   * the set of all elements of `s` that are not in `t`.
   */
def diff(s: Set, t: Set): Set = (x:Int) => contains(s, x) && !contains(t, x)
```

实现集合的过滤函数

```scala
/**
   * Returns the subset of `s` for which `p` holds.
   */
def filter(s: Set, p: Int => Boolean): Set = (x: Int) => contains(s, x) && p(x)
```

判断集合中的所有元素是否满足条件

```scala
/**
   * The bounds for `forall` and `exists` are +/- 1000.
   */
val bound = 1000

/**
   * Returns whether all bounded integers within `s` satisfy `p`.
   */
def forall(s: Set, p: Int => Boolean): Boolean = {
  def iter(a: Int): Boolean = {
    if (contains(x, a) && !p(a)) false
    else if (a > bound) true
    else iter(a + 1)
  }
  iter(-bound)
}
```

这里集合的大小是有限的，只能存放[-bound, bound]的数字，所以只要从`-bound`开始依次遍历即可。



判断集合中是否有某一个元素满足条件

```scala
  
/**
   * Returns whether there exists a bounded integer within `s`
   * that satisfies `p`.
   */
def exists(s: Set, p: Int => Boolean): Boolean = !forall(s, x => !p(x))
```



对集合中的每个元素做变换操作

```scala
/**
   * Returns a set transformed by applying `f` to each element of `s`.
   */
def map(s: Set, f: Int => Int): Set = (x: Int) => exists(s, (y: Int) => x == f(y))
```

这个实现是比较不那么容易理解

先看`exist(s, (y: Int) => x == f(y))`这个部分

对于旧的Set  s, 我们希望对其每个元素

新的Set t = {ne | e ∈ s,  ne = f(e)}

这个形式是不是和上面的很像？其实就是数学语言的一个转换

如果还是不明白，可以考虑代入具体的值，假如`s = 2 => true, f = (x: Int) => 2 * x`

`t = map(s, f)  = (x: Int) => exist(x, (y: int) => x == 2 * y)`

我们知道4是在`t`里面的

代入 `exist(4 , (y: Int) => 4 == 2 * y)`

而`s`中有元素2符合条件，所以上述式子的返回值为true。

验证完毕，完整代码可参照[此处](<https://gist.github.com/counter2015/986be53b64d6e6e1aa723651da27d91e>)

