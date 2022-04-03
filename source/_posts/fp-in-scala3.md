# Progfun 3.Data and Abstration

title: Progfun 3.Data and Abstration
date: 2019-09-04 01:32:30
tags: [scala, 读书笔记, Coursera] 
categories: [技术]

------





## 3.1 Class Hierarchies

以下有一个改编版的笑话

- OOP -> 你想要一个香蕉，结果得到的是一个森林，还有一个手里拿着香蕉的猴子
- FP -> 你想要一个香蕉，结果是先创造了一个宇宙，在这个宇宙初始化时，给新创建了个香蕉



> “面向对象编程语言的问题在于，它总是附带着所有它需要的隐含环境。你想要一个香蕉，但得到的却是一个大猩猩拿着香蕉，而其还有整个丛林。” — Joe Armstrong（Erlang语言发明人）



从下面的代码实现集合的合并操作

```scala
abstract class IntSet {
  def incl(x: Int): IntSet
  def contains(x: Int): Boolean
  def union(other: IntSet): IntSet
}

object Empty extends IntSet {
  def contains(x: Int): Boolean = false
  def incl(x: Int): IntSet = new NonEmpty(x, Empty, Empty)
  orerride def toString = "."
}

class NonEmpty(elem: Int, left: IntSet, right: IntSet) extends IntSet {
  def contains(x: Int): Booelan = 
    if (x < elem) left.contains(x)
    else if (x > elem) right.contains(x)
    else true
  
  def incl(x: Int): IntSet = 
    if (x < elem) new NonEmpty(elem, left.incl(x), right)
    else if (x > elem) new NonEmpty(elem, left, right.incl(x))
    else this
  
  override def toString = "{" + left + elem + right +"}"
}
```

结果如下

```scala
object Empty extends IntSet {
  def union(other: IntSet): IntSet = other
}

class NonEmpty(elem: Int, left: IntSet, right: IntSet) extends IntSet {
	def union(other: IntSet): IntSet = 
    ((left union right) union other) incl elem
}
```

- 空集合和其他集合合并，结果一定是另外一个集合
- 非空集合合并时，先将左节点和右节点合并，然后再将结果和右边的非空集合合并

函数式编程时有一个特点，就是使用递归，而递归又会带来一个问题，就是不好分析时间复杂度，这一点在后面作业的部分会再详细讲下。



## 3.2 How Classes Are Organized

```plain
if (true) 1 else false
```

上面这个表达式是什么类型的？

显然，1是`Int`, false是`Boolean`

也就是说，要找到一个类型，同时是它们两个的父类（base type）,给出如下选项

- Int
- Boolean
- AnyVal
- Object
- Any

显然答案是`AnyVal`，只要你打开REPL试下就知道了。

如果你注意到scala的类型系统

![](https://docs.scala-lang.org/resources/images/tour/unified-types-diagram.svg)

从这个图显而易见，是`AnyVal`

为什么不是`Any`?当然是取最近的一个，要不然凡是不一样的，结果都成`Any`了，`AnyVal`包含的信息比`Any`更多。

## 3.3 Polymorphism



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
  def head: Nothing = throw new NoSuchElementException("Nil.head")
  def tail: Nothing = throw new NoSuchElementException("Nil.tail")
}
```

加入给出这样一个`List`实现，要求实现一个函数`nth`，以整数`n`和`List[T]`类型的`xs`为参数，返回`List`中的第`n`个元素

元素从0开始编号，如果n超过了`List`的范围，抛出`IndexOutOfBoundsException`



```scala
def nth[T](n: Int, xs: List[T]): T = {
  if (n < 0 || xs.isEmpty) throw new IndexOutOfBoundsException
  if (n == 0) xs.head
  else nth(n-1, xs.tail)
}

scala> val list = new Cons(2, new Cons(3, new Nil))
list: Cons[Int] = Cons@65ef722a


scala> nth(0, list)
res2: Int = 2

scala> nth(1, list)
res3: Int = 3

scala> nth(2, list)
java.lang.IndexOutOfBoundsException
  at .nth(<console>:14)
  ... 32 elided

scala> nth(-1, list)
java.lang.IndexOutOfBoundsException
  at .nth(<console>:14)
  ... 32 elided

```





## 编程作业： Fucntional Sets

这一次的作业，需要基于推特的数据构建一个面向对象风格的`Set`

已经实现的函数如下

```scala
class NonEmpty (elem: Tweet, left: TweetSet, right: TweetSet) extends TweetSet {
  def contains(x: Tweet): Boolean =
    if (x.text < elem.text) left.contains(x)
    else if (elem.text < x.text) right.contains(x)
    else true

  def incl(x: Tweet): TweetSet = {
    if (x.text < elem.text) new NonEmpty(elem, left.incl(x), right)
    else if (elem.text < x.text) new NonEmpty(elem, left, right.incl(x))
    else this
  }

  def remove(tw: Tweet): TweetSet =
    if (tw.text < elem.text) new NonEmpty(elem, left.remove(tw), right)
    else if (elem.text < tw.text) new NonEmpty(elem, left, right.remove(tw))
    else left.union(right)

  def foreach(f: Tweet => Unit): Unit = {
    f(elem)
    left.foreach(f)
    right.foreach(f)
  }
  
}
```





```scala
class Empty extends TweetSet {
	def contains(tweet: Tweet): Boolean = false

  def incl(tweet: Tweet): TweetSet = new NonEmpty(tweet, new Empty, new Empty)

  def remove(tweet: Tweet): TweetSet = this

  def foreach(f: Tweet => Unit): Unit = ()
}
```

`Empty`中的函数实现比较简单，下文将只说明`NoEmpty`中对应函数的实现

### 1. 过滤

从上面的`NonEmpty.incl`的实现，可以看出，`TweetSet`的内部实现是一个按推文的内容排序的二叉树。

要求，实现`filter`和`filterAcc`,`filter`见下文代码

```scala
tweets.filter(tweet => tweet.retweets > 10)
```

`filterAcc`其实就是`filter`的结果和`acc`取一个并集。

自然地想到

```scala
def filter(p: Tweet => Boolean): TweetSet = 
  filterAcc(p, new Empty)
```

由于集合是有序的，可以参照之前小节中`List`的实现，写出以下代码

```scala
def filterAcc(p: Tweet => Boolean, acc: Tweet): TweetSet = 
  if (p(elem))
    left.filterAcc(p, right, filterAcc(p, acc.incl(elem)))
  else 
    left.filterAcc(p, right, filterAcc(p, acc))
```

这里直接借助已经实现的部分，来减少代码量。

怎么判断这个递归是一定能结束的呢？我们可以发现，每次调用该函数，都会处理当前节点的数据，然后在左边的分支上继续执行

也就是说，每次调用左分支上的元素数量减一，当左边为空时，调用的时`Empty.filterAcc`

显然有

```scala
class Empty extend TweetSet {
  def filterAcc(p: Tweet => Boolean, acc: Tweet): TweetSet = acc
}
```

### 2.合并

实现集合的合并操作

```scala
def union(that: TweetSet): TweetSet
```

这里是一个大坑，还记得之前提到的函数式编程计算算法复杂度的问题吗

```scala
def union(that: TweetSet): TweetSet = (left union right union that) incl elem
```

上面这个写法是没有问题的。

但是你要是不小心写成` left union (right union (that incl elem))`

等待你的是TLE(time limit exceed)

因为下面这种写法，会造成不必要的重复计算

具体分析可参见[课程讨论区](https://www.coursera.org/learn/progfun1/discussions/weeks/3/threads/AzJ-4CLYEeag6wpD-92Rcw/replies/NsyjCSMIEeaSbhIJy3C38Q/comments/NqN3gXxxEeagQhLtAgFLAw?page=2)

这个时间复杂度我不会算，但是据说是指数级别的

> **Warning : This method is a crucial part of the assignment. There are many ways to correctly code it, however some implementations run in an exponential time, so be careful, an inefficient implementation might result in a timeout during the grading process.**

### 3.排序

按照被转发的次数，对`TweetSet`中的推文排序，返回`TweetList`

```scala
def mostRetweeted: Tweet = {
    def max(a: Tweet, b: Tweet) : Tweet =
      if (a.retweets > b.retweets) a else b

    var res = elem
    if (!left.isEmpty) res = max(res, left.mostRetweeted)
    if (!right.isEmpty) res = max(res, right.mostRetweeted)
    res
  }

def descendingByRetweet: TweetList = 
  new Cons(mostRetweeted, remove(mostRetweeted).descendingByRetweet)
```

### 4. 组合上述功能

对所有包含`google`,`apple`关键词的推文进行过滤，收集后取并集，并按转发次数从高到低排序

```scala
object GoogleVsApple {
  val google = List("android", "Android", "galaxy", "Galaxy", "nexus", "Nexus")
  val apple = List("ios", "iOS", "iphone", "iPhone", "ipad", "iPad")

  lazy val googleTweets: TweetSet = allTweets.filter(p => google.exists(x => p.text.contains(x)))
  lazy val appleTweets: TweetSet = allTweets.filter(p => apple.exists(x => p.text.contains(x)))

  /**
    * A list of all tweets mentioning a keyword from either apple or google,
    * sorted by the number of retweets.
    */
  lazy val trending: TweetList = (googleTweets union appleTweets).descendingByRetweet
}
```

[完整代码](https://gist.github.com/counter2015/15ae7a7943cc00a424aae5bbd620868f)

