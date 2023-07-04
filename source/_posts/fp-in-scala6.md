# Progfun 6. Collections

title: Progfun 6.  Collections 
date: 2020-05-07 00:15:38
tags: [scala, 读书笔记, Coursera] 
categories: [技术]

------



## 6.1 Other Collections

这一小节没有练习

Vector的数据结构是一个32叉树，这是一个更加**中庸**的集合

这样访问的过程可以看成近似一个常数，因为`log_32(N)`

| N        | log_32(N) |
| -------- | --------- |
| 10       | 0.46      |
| 1000     | 1.38      |
| 100000   | 2.30      |
| 10000000 | 3.22      |

我们看到，N扩大100倍，对应的log_32(N)才大约增加1

另外一方面，对于每一Vector可以分治成32个更小的节点，这正好和现代缓存的大小接近(或许2020年来说是64更好一些？)，这样每一个都能充分例用到缓存，更加迅速。

一般来说，如果操作偏向于递归，那么List更适合；如果操作是批量的，比如`map/fold/filter`，那么Vector更快。

另外提一句，在2.13.2有神人[使用了新的数据结构来重构Vector](https://github.com/scala/scala/pull/8534)

这PR提得简直和范文一样，啧啧啧，比如这个图

![](https://user-images.githubusercontent.com/54262/68545584-cb77e500-03ce-11ea-99b0-c66f4ee3a037.png)

![](https://user-images.githubusercontent.com/54262/68545327-56a3ab80-03cc-11ea-9f31-fdbfc91b26f6.png)



据说使用到的新数据结构叫做`radix-balanced finger tree`

相关论文有空看看 <del>立了个Flag</del>

- https://infoscience.epfl.ch/record/169879/files/RMTrees.pdf
- http://www.staff.city.ac.uk/~ross/papers/FingerTree.html
-  http://comonad.com/reader/wp-content/uploads/2010/04/Finger-Trees.pdf 

## 6.2 Combinatorial Search and For-Expressions

考虑一个这样的问题

> 给出两个数组A,B，找出两个数组中对于 j < i, A_i + B_j为素数的所有(i, j)



```scala
val a = Array(2,3,4,4,5,6,6)
val b = Array(1,2,1,3,2,1,5)

def isPrime(i: Int): Boolean = {
  for (x <- 2 to i-1) 
    if (i % x == 0) return false
  true
}

for (i <- a; j <-b if j < i && isPrime(i+j)) yield (i, j)
```



## 6.3 Combinatorial Search Example

经典的N皇后问题

```scala
def queens(n: Int): Set[List[(Int, Int)]] = {
  def isSafe(tuple: (Int, Int), tuples: List[(Int, Int)]): Boolean = {
    tuples.forall(
      p =>
        p._1 != tuple._1 &&
          p._2 != tuple._2 &&
          (p._1 - tuple._1).abs != (p._2 - tuple._2).abs)
  }

  def placeQueens(k: Int): Set[List[(Int, Int)]] =
    if (k == 0)
      Set(List())
    else
      for {
        queens <- placeQueens(k - 1)
        column <- 0 until n
        queen = (k - 1, column)
        if isSafe(queen, queens)
      } yield queen :: queens

  placeQueens(n)
}

def show(queens: List[(Int,Int)]) = {
  val length = queens.length
  val res = Array.fill(length, length)("* ")
  for ((i, j) <- queens) {
    res(i)(j) = "X "
  }
  "\n" + res.map(_.mkString).mkString("\n")
}

print(queens(4) map show mkString "\n")
```



## 6.4 Maps

此map非彼map，为集合类里的map<key, value>

Martin写了一个用map来实现多项式结构的例子

```scala
class Poly (terms0: Map[Int, Double]) {
  def this(bindings: (Int, Double)*) = this(bindings.toMap)
  val terms = terms0 withDefaultValue 0.0
  def +(other: Poly) = new Poly(terms ++ (other.terms map adjust))
  def adjust(term: (Int, Double)): (Int, Double) = {
    val (exp, coeff) = term
    exp -> (coeff + terms(exp))
  }
  
  override def toString = 
    (for ((exp, coeff) <- terms.toList.sorted.reverse) yield coeff+"x^"+exp) mkString " + "
}

val p1 = new Poly(1 -> 2.0, 3 -> 4.0, 5 -> 6.2)
val p2 = new Poly(0 -> 3.0, 3 -> 7.0)
p1 + p2
```

这里的习题是使用`foldLeft`版本来实现`+`方法，对应的函数签名如下

```scala
def +(other: Poly) = new Poly((other.terms foldLeft ???)(addTerm))
def addTerm(terms: Map[Int, Double], term: (Int, Double)) = ???
```

实现如下

```scala
class Poly (terms0: Map[Int, Double]) {
  def this(bindings: (Int, Double)*) = this(bindings.toMap)
  val terms = terms0 withDefaultValue 0.0
  def +(other: Poly) =  new Poly((other.terms foldLeft terms)(addTerm))
  
  def addTerm(terms: Map[Int, Double], term: (Int, Double)) = {
    val (exp, coeff) = term
    terms + (exp -> (coeff + terms(exp)))
  }
  
  def adjust(term: (Int, Double)): (Int, Double) = {
    val (exp, coeff) = term
    exp -> (coeff + terms(exp))
  }
  
  override def toString = 
    (for ((exp, coeff) <- terms.toList.sorted.reverse) yield coeff+"x^"+exp) mkString " + "
}
```

对比这两个实现，哪一个性能更好呢？Martin认为`foldLeft`版本的更快，

从`addTerm`和`adjust`来看第一个用的语句更少，但实际上却是后者更快

这是因为两者的调用方式不同

```scala
def +(other: Poly) = new Poly(terms ++ (other.terms map adjust))
def +(other: Poly) = new Poly((other.terms foldLeft terms)(addTerm))
```

> So, I would argue the one with foldLeft is more efficient because what happens 
>
> here is that each of these bindings will be immediately added to our terms Maps so, 
>
> we build up the result directly, whereas before, 
>
> we would create another list of terms that contain the adjusted terms and 
>
> then we would concatenate this list to the original one.

前者会把terms做map操作，新生成的临时`map`再和原来的做连接，而后者省去了中间生成的步骤



口说无凭，我们来上测试

这里用的是[sbt-jmh插件](https://github.com/ktoso/sbt-jmh)

简单的使用可以参看这篇[文章](https://mp.weixin.qq.com/s/WN50R5alQg1ATl8yWlmm9A)

随便写了点测试代码,测试环境为

- Scala 2.13.2
- sbt 1.13.10

为什么包名叫abc呢，不放包里会报NPE错误，但是我又不擅长起名字

相关issue:  https://github.com/ktoso/sbt-jmh/issues/134 

```scala
package abc

import java.util.concurrent.TimeUnit

import org.openjdk.jmh.annotations._

@State(Scope.Benchmark)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@BenchmarkMode(Array(Mode.Throughput))
class PolyTest {
  val p1 = new Poly(1 -> 2.0, 3 -> 4.0, 5 -> 6.2)
  val p2 = new Poly(0 -> 3.0, 3 -> 7.0)
  val p3 = new Poly(0 -> 3.0, 1 -> 2.0, 3 -> 11.0, 5 -> 6.2)

  @Benchmark
  @OperationsPerInvocation
  def mytest1() = {
    p1 add1 p2
  }

  @Benchmark
  @OperationsPerInvocation
  def mytest2() = {
    p1 add2 p2
  }
}

object PolyTest {
  val OperationsPerInvocation = 10000 * 2
}
```

测试依赖的代码

```scala
package abc

class Poly(terms0: Map[Int, Double])  {
  def this(bindings: (Int, Double)*) = this(bindings.toMap)
  val terms: Map[Int, Double] = terms0 withDefaultValue 0.0

  def add1(other: Poly) = new Poly(terms ++ (other.terms map adjust))
  def add2(other: Poly) =  new Poly((other.terms foldLeft terms)(addTerm))

  def addTerm(terms: Map[Int, Double], term: (Int, Double)): Map[Int, Double] = {
    val (exp, coeff) = term
    terms + (exp -> (coeff + terms(exp)))
  }

  def adjust(term: (Int, Double)): (Int, Double) = {
    val (exp, coeff) = term
    exp -> (coeff + terms(exp))
  }

  override def toString: String =
    (for ((exp, coeff) <- terms.toList.sorted.reverse) yield coeff+"x^"+exp) mkString " + "
}
```

跑的过程中风扇狂转，IDEA跑的结果还有乱码，改到wsl开sbt跑

测试结果如下

```scala
sbt > jmh:run -i 3 -wi 3 -f3 -t2 .*mytest.*

...
[info] # Run complete. Total time: 00:06:04
[info] REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
[info] why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
[info] experiments, perform baseline and negative tests that provide experimental control, make sure
[info] the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
[info] Do not assume the numbers tell you what you want them to tell.
[info] Benchmark          Mode  Cnt   Score   Error   Units
[info] PolyTest.mytest1  thrpt    9   2.979 ± 0.564  ops/us
[info] PolyTest.mytest2  thrpt    9  16.595 ± 4.234  ops/us
[success] Total time: 366 s (06:06), completed May 5, 2020 3:12:31 PM
```

差距还是挺明显的，也验证了之前的说法。

## 6.5 Putting the Pieces Together

以下的例子来源于

`Lutz Prechelt: An Empirical Comparison of Seven Programming Languages. IEEE Computer 33(10): 23-29 (2000)`

标题说的七种语言分别是

- TCL
- Python
- Perl
- Rexx
- Java
- C++
- C

对于上述语言编写这个例子所需要的代码行数

- 脚本语言：100行
- 其他语言：200 ~ 300 行

这一小节结合之前的内容，实现一个简单的程序，把电话号码(九键字母)转换成单词

![](https://counter2015.com/picture/phone-keyboard.jpg)

假设我们有了这么一个字典

```scala
val mnemonics = Map(
  '2' -> 'ABC',
  '3' -> 'DEF',
  '4' -> 'GHI',
  '5' -> 'JKL',
  '6' -> 'MNO',
  '7' -> 'PQRS',
  '8' -> 'TUV',
  '9' -> 'WXYZ'
)
```

设计一个函数如`translate(phoneNumber)`，输出可以拼写出来的单词

如`translate("7225247386")`的输出结果中应该包括`Scala is fun`

除了给出对应的字典外，当然还给出了单词表

```scala
val in = Source.fromURL("http://lamp.epfl.ch/files/content/sites/lamp/files/teaching/progfun/linuxwords")
```

不幸的是，这个网址已经失效了

好在还能通过后面的编程作业`Anagrams`拿到同名文件`linuxwords`

给出的模板 代码如下

```scala
import scala.io.Source

object x {
  val in = Source.fromFile("linuxwords.txt")
  val words = in.getLines.toList
  
  val mnem =  Map(
  '2' -> 'ABC', '3' -> 'DEF',
  '4' -> 'GHI', '5' -> 'JKL', '6' -> 'MNO',
  '7' -> 'PQRS', '8' -> 'TUV', '9' -> 'WXYZ')
  
  // Invert the mnem map to give a map from chars 'A' ... 'Z' to '2' ... '9'
  val charCode: Map[Char, Char] = ???
  
  // Maps a word to the digit string it can represent, e.g. "Java" -> "5282"
  def wordCode(word: String): String = ???
  
  /*
    a map from digit strings to the words that represent them,
    e.g. "5282" -> List("Java", "Kata", "Lava", ...)
    Note: A missing number should map to empty set, e.g. "1111" -> List()
  */
  def wordsForNum: Map[String, Seq[String]] = ???
  
  // Return all ways to encode a number as a list of words
  def encode(number: String): Set[Seq[String]] = ???
}
```

有了样板，填空还是很容易的

```scala
import scala.io.{BufferedSource, Source}

object x {
  val in: BufferedSource = Source.fromFile("linuxwords.txt")
  val words: List[String] = in.getLines.toList.filter(word => word forall (ch => ch.isLetter))

  val mnem =  Map(
    '2' -> "ABC", '3' -> "DEF",
    '4' -> "GHI", '5' -> "JKL", '6' -> "MNO",
    '7' -> "PQRS", '8' -> "TUV", '9' -> "WXYZ")

  val charCode: Map[Char, Char] = {
    for ((digit, str) <- mnem; ch <- str) yield ch -> digit
  }

  def wordCode(word: String): String = word.toUpperCase.map(charCode)

  def wordsForNum: Map[String, Seq[String]] =
    words.groupBy(wordCode).withDefaultValue(Seq())

  def encode(number: String): Set[List[String]] = {
    if (number.isEmpty) Set(List())
    else {
      (for {
        split <- 1 to number.length
        word <- wordsForNum(number take split)
        rest <- encode(number drop split)
      } yield word :: rest).toSet
    }
  }
}
encode("7225247386")
```

简单说下思路吧

首先从当前目录的`linuxwords.txt`读取我们需要的单词表，这个文件可以在后面的作业中的`src/main/resources/forcomp/linuxwords.txt`找到，每行一个单词，大约有4万行。

读取进来后做一个简单的过滤，因为码表中只能映射字母，所以要过滤掉包含非字母的单词。

```scala
scala> words.filter(s => s.exists(ch => !ch
     | .isLetter))
val res3: List[String] = List(Bhagavad-Gita, home-brew, Ibero-, L'vov, Modula-2, Modula-3, O'Brien, O'Connell, O'Connor, O'Dell, O'Donnell, O'Dwyer, O'Hare, O'Leary, O'Neill, O'Shea, O'Sullivan, Serbo-, Sino-, Uruguay'a)
```

`mnem`作为映射表，就硬编码了，但是对应的倒排表不应该再次硬编码，这里简单的复用了下

```scala
val charCode: Map[Char, Char] = {
  for ((digit, str) <- mnem; ch <- str) yield ch -> digit
}
```

拿出k,v，对于v迭代，每个字母作为新的k', 原来的k作为v',构造(k' -> v')的map

如果你在 exercism.io 上刷题，你会发现这个步骤和ETL的题面非常相似

下面`wordCode`函数，将字符转换成大写后，调用map函数，传入的参数为Map（是不是像绕口令)

前面的map是变换操作，类似于filter，后面的Map是一个collection，类似于List

举个例子

```scala
scala> val m = Map(1 -> 2) withDefaultValue 3
val m: scala.collection.immutable.Map[Int,Int] = Map(1 -> 2)

scala> List(1,2,3).map(m)
val res4: List[Int] = List(2, 3, 3)
```

`wordsForNum`是把一个数字输入的字符串，映射成所有的合法单词序列的集合

`encode`巧妙地使用了`for`，从外到内依次为

- 遍历分割点
- 对于分割点之前的输入序列，能构成的所有单词集合进行遍历
- 对剩余部分递归的求出结果

将递归求出的结果和分割点前半部分的结果连接，作为一个合法的输出，把所有的输出收集起来，再转换成我们需要的Set格式

把上述代码在REPL中运行的结果如下

```scala
scala> import scala.io.{BufferedSource, Source}
import scala.io.{BufferedSource, Source}

scala> val in: BufferedSource = Source.fromFile("linuxwords.txt")

val in: scala.io.BufferedSource = <iterator>

scala>   val words: List[String] = in.getLines.toList.filter(word => word forall (ch => ch.isLetter))
val words: List[String] = List(Aarhus, Aaron, Ababa, aback, abaft, abandon, abandoned, abandoning, abandonment, abandons, abase, abased, abasement, abasements, abases, abash, abashed, abashes, abashing, abasing, abate, abated, abatement, abatements, abater, abates, abating, Abba, abbe, abbey, abbeys, abbot, abbots, Abbott, abbreviate, abbreviated, abbreviates, abbreviating, abbreviation, abbreviations, Abby, abdomen, abdomens, abdominal, abduct, abducted, abduction, abductions, abductor, abductors, abducts, Abe, abed, Abel, Abelian, Abelson, Aberdeen, Abernathy, aberrant, aberration, aberrations, abet, abets, abetted, abetter, abetting, abeyance, abhor, abhorred, abhorrent, abhorrer, abhorring, abhors, abide, abided, abides, abiding, Abidjan, Abigail, Abilene, ...

scala> val mnem =  Map(
     |     '2' -> "ABC", '3' -> "DEF",
     |     '4' -> "GHI", '5' -> "JKL", '6' -> "MNO",
     |     '7' -> "PQRS", '8' -> "TUV", '9' -> "WXYZ")
val mnem: scala.collection.immutable.Map[Char,String] = HashMap(8 -> TUV, 4 -> GHI, 9 -> WXYZ, 5 -> JKL, 6 -> MNO, 2 -> ABC, 7 -> PQRS, 3 -> DEF)

scala>   val charCode: Map[Char, Char] = {
     |     for ((digit, str) <- mnem; ch <- str) yield ch -> digit
     |   }
val charCode: Map[Char,Char] = HashMap(N -> 6, T -> 8, U -> 8, F -> 3, A -> 2, M -> 6, I -> 4, B -> 2, P -> 7, C -> 2, H -> 4, W -> 9, O -> 6, D -> 3, E -> 3, X -> 9, Y -> 9, J -> 5, G -> 4, V -> 8, Q -> 7, L -> 5, K -> 5, R -> 7, Z -> 9, S -> 7)

scala> def wordCode(word: String): String = word.toUpperCase.map(charCode)
def wordCode(word: String): String

scala>  def wordsForNum: Map[String, Seq[String]] =
     |     words.groupBy(wordCode).withDefaultValue(Seq())
def wordsForNum: Map[String,Seq[String]]

scala> def encode(number: String): Set[List[String]] = {
     |     if (number.isEmpty) Set(List())
     |     else {
     |       (for {
     |         split <- 1 to number.length
     |         word <- wordsForNum(number take split)
     |         rest <- encode(number drop split)
     |       } yield word :: rest).toSet
     |     }
     |   }
def encode(number: String): Set[List[String]]

scala> encode("7225247386")
val res1: Set[List[String]] = HashSet(List(rack, ah, re, to), List(sack, ah, re, to), List(Scala, ire, to), List(rack, bird, to), List(pack, air, fun), List(pack, ah, re, to), List(pack, bird, to), List(Scala, is, fun), List(sack, bird, to), List(sack, air, fun), List(rack, air, fun)) 
                               
scala> def translate(number: String): Set[String] = encode(number) map (_ mkString " ")
def translate(number: String): Set[String]

scala> translate("7225247386")
val res2: Set[String] = HashSet(pack bird to, Scala ire to, Scala is fun, rack ah re to, pack air fun, sack air fun, pack ah re to, sack bird to, rack bird to, sack ah re to, rack air fun)                   
```

和上面提到的七种语言对比，Scala的行数是最少的，只用了50行。

## 编程作业：Anagrams
作业下载地址：http://alaska.epfl.ch/~dockermoocs/progfun1/forcomp.zip

 anagram (异序词)是把单词重新排列后得到的新的单词，比如

对于句子 `I love you` 可以改写成`You  I love`的异序词，也可以改写成`You olive`

显然，这里是不区分大小写的

现在需要写出一个函数

```scala
type Word = String
type Sentence = List[Word]
def sentenceAnagrams(sentence: Sentence): List[Sentence]
```

能输出一个句子的所有异序词转换方案

这个问题其实就是上一个问题的变种，照猫画虎就行了。

核心代码

```scala
  def sentenceAnagrams(sentence: Sentence): List[Sentence] = {
    def helper(occurrences: Occurrences): List[Sentence] = occurrences match {
      case Nil => List(Nil)
      case _ =>
        for (comb <- combinations(occurrences) if comb.nonEmpty;
             word <- dictionaryByOccurrences.getOrElse(comb, Nil);
             sentence <- helper(subtract(occurrences, wordOccurrences(word)) ) )
          yield word :: sentence 

    }

    helper(sentenceOccurrences(sentence))
  }
```

这个做法有一个问题，就是我们总是在不停地重复计算已经知道的结果，如果有办法保存下来就好了

那么如何来做持久化呢？

参考以下文章，我简单粗暴地改造了下

-  https://medium.com/musings-on-functional-programming/scala-optimizing-expensive-functions-with-memoization-c05b781ae826 
-  https://stackoverflow.com/questions/16257378/is-there-a-generic-way-to-memoize-in-scala 

```scala
  def sentenceAnagrams(sentence: Sentence): List[Sentence] = {
    val cache = collection.mutable.Map.empty[Occurrences, List[Sentence]]
    def helper(occurrences: Occurrences): List[Sentence] = occurrences match {
      case Nil => List(Nil)
      case _ =>
        for (comb <- combinations(occurrences) if comb.nonEmpty;
             word <- dictionaryByOccurrences.getOrElse(comb, Nil);
             remain = subtract(occurrences, wordOccurrences(word));
             sentence <- cache.getOrElse(remain, {
               cache update (remain, helper(remain))
               helper(remain)
             }))
          yield word :: sentence
    }
    helper(sentenceOccurrences(sentence))
  }
```

这就不跑jmh了，电脑实在是吃不消


## 总结

之前的文章

- [Progfun 1.Getting Start](https://counter2015.com/2019/07/29/fp-in-scala1/)
- [Progfun 2.High Order Fuctions](https://counter2015.com/2019/08/03/fp-in-scala2/)
- [Progfun 3.Data and Abstration](https://counter2015.com/2019/09/04/fp-in-scala3/)
- [Progfun 4. Types and Pattern Matching](https://counter2015.com/2019/09/24/fp-in-scala4/)
- [Progfun 5. Lists](https://counter2015.com/2020/03/09/fp-in-scala5/)

终于写完了第一个系列的笔记，说实话这个其实老早就把作业写完了，但是有些不懂的地方，所以看了第二遍

其实第二遍还有些没懂的，比如泛型的协变逆变不变为什么要这样设计

年末就要发Scala 3了，这算是先把系统学习Scala 2的坑填了。

Martin讲课优点是非常适合新手，不是直接给你一个完整的解决方案，而是向你展示一段段代码是按什么样的思路写出来的，并且不断的进行重构，同时还会演示常见的出错场景和如何解决。

作业的习题难度，习惯后也还好，因为有完善的函数签名和注释，写起来就像做填空题一样

Martin讲课有个缺点就是，完全的平铺直叙，语调不带变的<del>无情的朗读机器</del>

字幕有时候会打错，稍微有点影响理解，不过课程整体感觉还是挺好的，我会给85/100的分

### What's more?

- FP（Functional Programming） design
- FP and state
- 并行编程和分布式系统
- Spark

