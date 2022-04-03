# Progfun 1.Getting Start

title: Progfun 1.Getting Start
date: 2019-07-29 23:59:30
tags: [scala, 读书笔记, Coursera] 
categories: [技术]
mathjax: true

------

![](<http://counter2015.com/picture/profun-1-1.jpg>)

## 前言

Scala Puzzle系列要暂时停更一段时间了，之前我以为Scala Puzzle是一本趣味读物，结果发现是进阶读物，做到一半发现做不下去了（摔

回过头来重新系统学习Scala基础知识，这里选择遵循[Scala官网的课程列表](<https://www.scala-lang.org/blog/2019/05/21/news-from-the-scala-moocs.html>)

第一门课程在coursea上叫[Functional Programming Principles in Scala](https://www.coursera.org/learn/progfun1)，由Martin Odersky主讲,课程的主要语言为英文，提供字幕。

这个系列的笔记名称就叫"Progfun"，因为原文“Scala 函数式程序设计原理”是在太长

主要按周为进度记录每周的视频中的练习和留下的课后习题部分。

## 环境准备

在第一周的课程材料里有详细的环境搭建过程说明，这里就不再赘述了。

我使用的开发环境是Windows + Intellij IDEA + WSL + XShell + SBT

本系列使用的REPL版本为 Scala 2.11.12(当前时间最新的Scala版本为2.13.0)

## Week 1: Getting Start , Function & Evaluation

第一周的课程视频过程约1小时20分钟，讲了下Scala和其他语言的对比和特性

```scala
def loop: Int = loop
```

### 1.2 Element of Programming

对于同样的表达式，有两种计算方法

- call by value
- call by name

call by value的好处是对于每一个函数的参数，只用进行一次计算

call by name的好处是对于函数体内没有使用到的参数不用进行计算

这样说有些抽象，让我们看下下面的例子

假设你有一个函数`def test(x: Int, y: Int) = x * x`

对于下面几种函数参数的调用

- test(2,3)
- test(3+4,8)
- test(7,2*4)
- test(3+4,2*4)

是call by value更快，call by name更快，还是两者一样快（这里把加法和乘法看作一样快）？

我们先来看第一个例子test(2,3)

call by value :

-  test(2,3) = 2 * 2 = 4

call by name:

- test(2,3) = 2 * 2 = 4

可以看到，在最简单的情况下，两者的步骤是一样多的。

对于第二个例子 test(3+4,8)

call by value:

- test(3+4,8) = test(7, 8) = 7 * 7 = 49

call by name:

- test(3+4,8) = (3+4) * (3+4) = 7 * (3+4) = 7 * 7 = 49

这里就体现出差距了，call by value由于在参数进入是就做了计算，所以只用了三步，而call by name则直接把参数传了下去，需要四步。

再来看第三个例子 test(7,2*4)

call by value:

- test(7,2 * 4) = test(7,8) = 7 * 7 =49

call by name:

- test(7,2 * 4) = 7 * 7 = 49

这里由于第二个参数不参与函数体内的计算，call by name反而更胜一筹

最后一个例子 test(3+4, 2*4)

call by value:

- test(3+4, 2 * 4) = test(7, 2 * 4) = test(7, 8) = 7 * 7 = 49

call by name:

- test(3+4, 2 * 4) = (3+4) * (3+4) = 7 * (3+4) = 7 * 7 = 49

最后还是使用了相同的步骤数。

### 1.3 Evaluation Strategies and Termination

大多数情况下，call by value的计算效率比call by name高，因为它避免了传入参数的重复计算，

但是，对于call by name 能终止的条件，call by value可能会一直执行，这也很好理解，因为call by value需要计算

出所有的参数值，比如

``` scala
def loop: Int = loop

def constOne(x: Int, y: => Int) = 1
```

注意，这里 `y: => Int`里，不能把冒号和等号连写，要不然会被识别成标识符

```scala
scala> def constOne(x: Int, y:=> Int) = 1

<console>:1: error: ':' expected but identifier found.
def constOne(x: Int, y:=> Int) = 1

```

`y: => Int`这样的写法，是为了指明对于参数`y`用call by name的方式来使用

那么，对于以上的函数，下面两个调用的结果如何呢？

- constOne(1+2, loop)
- constOne(loop, 1+2)



constOne(1+2, loop) :

- constOne(1+2, loop) = constOne(3, loop) = 1

constOne(loop, 1+2):

- constOne(loop, 1+2) = constOne(loop, 1+2) = ....

你会发现，第二种写法造成了死循环

如果对上面的例子还不太明白，下面有个直观的[样例](<https://stackoverflow.com/questions/13337338/call-by-name-vs-call-by-value-in-scala-clarification-needed>)

```scala
scala> def something() = {
     |   println("calling something")
     |   1 // return value
     | }
something: ()Int

scala> def callByValue(x: Int) = {
     |   println("x1=" + x)
     |   println("x2=" + x)
     | }
callByValue: (x: Int)Unit

scala> 

scala> def callByName(x: => Int) = {
     |   println("x1=" + x)
     |   println("x2=" + x)
     | }
callByName: (x: => Int)Unit

scala> callByValue(something())
calling something
x1=1
x2=1

scala> callByName(something())
calling something
x1=1
calling something
x2=1

```

### 1.4 Conditions and Value Definitions

```scala
// and(x,y) == x && y
// 要求不用 && 或 || 来实现这个函数

def and(x: Boolean, y: => Boolean) = if (!x) false else y
```

这里由于 && 的短路机制，即如果x的值 为真，就不需要再计算y的值，因而y的传参使用的是call by name的方法。

也可以写成`def and(x: Boolean, y: => Boolean) = if (x) y else false`

类似的，`or`函数的实现自然也能写出来

```scala
def or(x: Boolean, y: => Boolean) = if (x) true else y
```

### 1.5 Example: square roots with Newton's method

以下是对牛顿迭代法求平方根的一个简易解释

首先我们选定一个值作为初始条件，如y = 1

对于求sqrt(2), 2/y = 2, 如果我们求解的y = sqrt(x), 那么 x / y = y

这里可以看出1<2我们估计的值不对，为了下一次估计更加准确，我们取 (y + x /y ) /2 作为下一次估计值

这里y2 = (1 + 2)/2 = 1.5 2/1.5=1.3333

可以发现这里估计的更准了，因为 x/y 与y的差值更小了，这里可以证明，重复这个步骤，结果是一个收敛的数列（证明略）。

那么，只要我们的估计值 |y - x/y| < e , e为一个足够小的数，如0.0001，我们就能进近似的认为，我们现在估计的y值为sqrt(2)

实现如下



```scala
object session {
  def abs(x: Double) = if (x < 0) -x else x
  
  def sqrtIter(guess: Double, x: Double): Double = 
    if (isGoodEnough(guess, x)) guess
    else sqrtIter(imporve(guess, x), x)
  
  def isGoodEnough(guess: Double, x: Double) =
    abs(guess - x / guess) < 0.001 * guess
  
  def imporve(guess: Double, x: Double) = 
    (guess + x / guess) / 2
  
  def sqrt(x: Double) = sqrtIter(1.0, x)
  
}
```
看起来似乎没啥问题
```scala
scala> session.sqrt(2)
res0: Double = 1.4142156862745097

scala> res0 * res0 
res1: Double = 2.0000060073048824
```

然而并不是，Martin接下来立刻就丢过来三个问题

1. `isGoodEnough`函数对于非常小的数字不太准确，对于很大的数字看起来又无法终止，能解释原因吗
2. 重新实现`isGoodEnough`函数以规避上述问题
3. 测试以下样例
   - 0.0001
   - 0.1e-20
   - 1.0e20
   - 1.0e50



1. 对比较小的数据不准确是因为`isGoodEnough`用的是绝对误差,可对该函数做简单修改，以输出中间值

   ```scala
     def isGoodEnough(guess: Double, x: Double) =
     {println(guess, x/guess);abs(guess * guess - x) < 0.001}
   ```

   运行结果为

   ```scala
   scala> session.sqrt(1e-6)
   (1.0,1.0E-6)
   (0.5000005,1.999998000002E-6)
   (0.250001249999,3.999980000116E-6)
   (0.12500262498950004,7.999832004199894E-6)
   (0.06250531241075212,1.5998640138433748E-5)
   (0.031260655525445276,3.198909246116188E-5)
   res2: Double = 0.031260655525445276
   
   ```

   这里 0.03 * 0.03 - 0.0001 < 0.001但是显然最后的结果不好

   我们当然可以直接把0.001改成一个更小的值，但更好的一个做法是用相对误差，修改后的函数如下

   ```scala
   def isGoodEnough(guess: Double, x: Double) = 
     abs(guess - x / guess) < 0.001
   ```

   修改后的结果看起来更正常了

   ```scala
   scala> session.sqrt(1e-6)
   res3: Double = 0.0012961915927068783
   
   scala> session.sqrt(1e50)
   //REPL 会卡住
   
   ```

   对应地，对于较大的输入的数，它根本不会收敛到终止条件。

   这是因为double 类型浮点数，在IEEE 754标准中，用64位来存储，其中分配了52位来存储浮点数的有效数字，11位存储指数，1位存储正负号

   在二进制表示中，有效数字大于52位的部分会舍去

   ```scala
   
   scala> def log2 = (x:Double) => Math.log(x)/Math.log(2.0)
   log2: Double => Double
   
   scala> log2(1e50)
   res1: Double = 166.09640474436813
   
   
   ```

   这里可以看到1e60转换成二进制有近166位，而浮点数的精度只有52位，也就是说，最小误差也是`2^114`远大于0.001,所以自然不会到达终止条件

2. 基于以上的分析，我们重新实现的函数如下

   ```scala
   object session {
     val eps = 0.001
     
     def abs(x: Double) = if (x < 0) -x else x
     
     def sqrtIter(guess: Double, x: Double): Double = 
       if (isGoodEnough(guess, x)) guess
       else sqrtIter(imporve(guess, x), x)
     
     def isGoodEnough(guess: Double, x: Double) =
       abs(guess - x / guess) < eps * guess
     
     def imporve(guess: Double, x: Double) = 
       (guess + x / guess) / 2
     
     def sqrt(x: Double) = sqrtIter(1.0, x)
     
   }
   ```

3. 运行测试

   ```scala
   scala> session.sqrt(0.0001)
   res3: Double = 0.010000714038711746
   
   scala> session.sqrt(1e20)
   res4: Double = 1.0000021484861237E10
   
   scala> session.sqrt(1e-20)
   res6: Double = 1.0000021484861236E-10
   
   scala> session.sqrt(1e50)
   res7: Double = 1.0000003807575104E25
   ```

   





### 1.6 Blocks and  Lexical Scope

下面这段代码的运行结果是什么

```scala
val x = 0

def f(y： Int) = y + 1

val result = {
  val x = f(3)
  x * x
} + x
```



要理解这段代码，首先要理解变量的作用域，对于变量`x`来说，它在这段代码中被定义了两次，在外部的值为0，而在result的内部，x = f(3) = 4,result代码块返回的结果`x*x`为16，因而result = 16 + 0 = 16

### 1.7 Tail Recursion

设计一个尾递归的阶乘函数

```scala
def factorial(n: Int): BigInt = {
  def loop(result: BigInt, n: Int): BigInt =
    if (n == 0) result
    else loop(result * n, n - 1)
  loop(1, n)
}

scala> factorial(233)
res0: BigInt = 96880983124035637644628191427119032733410705766804841418415225561765762804210624629481224430320029511142586282867348560105183527088477325371186685286160118876894338289245202198092574546071969642723246616617867678202922360665112163722099321474343230404599030924937888027299552398154265468854355637510534924861247309970389790443978423417096800652347264383303351164354900418398161431781398827918950400000000000000000000000000000000000000000000000000000000

```






### Example Assignment

这一步之前，首先需要配置好运行环境，这里就不赘述了。

将课程提供的`example.zip`下载到本地解压，用Intellij打开

我们需要完成的就是src/main/scala/example/Lists.scala这个文件

> Notes: 注意不要修改已经实现的方法或名称

从IDE中打开sbt shell，首次运行需要下载依赖包，这可能会花费一段时间，长短视网络情况而定。

```scala
> console

scala> import example.Lists._
import example.Lists._

scala>  max(List(1,3,2))
scala.NotImplementedError: an implementation is missing
  at scala.Predef$.$qmark$qmark$qmark(Predef.scala:230)
  at example.Lists$.max(Lists.scala:41)
  ... 42 elided
```

这里报错因为这个方法尚未被实现，这个作业的任务就是这个了，实现List的sum和max方法

这两个方法的详细说明，注释里已经解释地很清楚了，这里就不在赘述。

```scala
def sum(xs: List[Int]): Int = {
  def reduceLeft(sum:Int, xs: List[Int]): Int = {
    if (xs.isEmpty)
      sum
    else
      reduceLeft(sum + xs.head, xs.tail)
    }

  reduceLeft(0, xs)
}

def max(xs: List[Int]): Int = {
  if (xs.isEmpty) throw new java.util.NoSuchElementException

    def reduceLeft(max:Int, xs: List[Int]): Int = {
      if (xs.isEmpty)
        max
      else
        reduceLeft(Math.max(max,xs.head), xs.tail)
    }

    reduceLeft(Int.MinValue, xs)
}
```

这里用递归的方式实现



这里还需要写单元测试来检验我们的答案。

对应的文件为`src/test/scala/example/ListSuite`

该文件内有详细的说明编写单元测试的注意事项，你需要对里面的代码进行修改，使得程序在sbt shell中运行`test`命令时，能顺利通过。

正确通过时如下所示

```scala
> test

[IJ]> test
[info] ListsSuite:
[info] - one plus one is two
[info] - one plus one is three?
[info] - details why one plus one is not three
[info] - intNotZero throws an exception if its argument is 0
[info] - sum of a few numbers
[info] - max of a few numbers
[info] - sum of a few negative numbers
[info] - sum of zeros
[info] - max of null List show throw NoSuchElementException
[info] Run completed in 1 second, 54 milliseconds.
[info] Total number of tests run: 9
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 9, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[success] Total time: 3 s, completed 2019-7-17 21:52:55
```

具体修改步骤就不列出了。

最后完成时记得提交，提交步骤也是在sbt shell里运行，在作业的网页右侧可以生成识别码

在sbt shell中运行`submit <email-address> <your-code>`就可以提交

如`submit 123456789@gmail.com HEmio6RWZ7ribfh1`

之后你可以在网页中查看自己的分数，至此示例作业就完成了。



### 作业： Recursion

1. 帕斯卡三角（杨辉三角）

   ```
   1
   1 1
   1 2 1
   1 3 3 1
   1 4 6 4 1
   ```

   pascal(0,2) = 1, pascal(1,2) = 2, pascal(1,3) = 3

   求paccal(col, row)的值，col和row都是从0开始计算

   如果你还记得二次项系数（二项式定理），那么就好办了
   $$
   若 C^m_n = \frac{n!m!}{(n-m)!} ,n >= m 且 m,n∈N^+
   $$
   那么有
   $$
   C_n^m = C_{n-1}^{m-1} + C_n^{m-1}
   $$
   

   已知
   $$
   C_n^0 = C_n^n = 1
   $$
   

   代码实现如下

   ```scala
   def pascal(c: Int, r: Int): Int = 
     if (c == 0 || c == r) 1
     else pascal(c - 1, r - 1) + pascal(c, r - 1)
   ```

   

2. 判断括号是否平衡

   判断一串字符串文本中包含的括号是否“平衡”

   例如，以下序列为平衡

   - (if (zero? x) max (/ 1 x))
   - I told him (that it’s not (yet) done). (But he wasn’t listening)

   以下序列为不平衡

   - :-)
   - ())(

   简单来说 可以总结为如下规则

   - 空串或者不含括号的字符串为平衡的
   - 如果R为平衡的，那么(R)也是平衡的
   - 如果R为平衡的，S为平衡的，那么RS也是平衡的

   代码实现如下

   ```scala
   def balance(chars: List[Char]): Boolean = {
     def balanced(chars: List[Char], count: Int): Boolean = {
       if (chars.isEmpty) count == 0
         else if (chars.head == '(') balanced(chars.tail, count + 1)
         else if (chars.head == ')') count > 0 && balanced(chars.tail, count - 1)
         else balanced(chars.tail, count)
     }
   
     balanced(chars, 0)
   }
   ```

   思路是，一个平衡的序列，`(`和`)`数量必须相同且合法。

   - 空串是合法的序列
   - 如果字符串的第一个字符为`(`， 当前计数+1，递归执行剩下的字符串
   - 如果字符串的第一个字符为`)`， 且当前计数大于零，那么递归执行剩下的字符串，当前计数-1
   - 如果不符合以上条件，递归执行剩下的字符串

3. 计算找零

   给你一定数目的钱`money`，问用`coins`（一个整型数组）中的零钱，有多少种方法能凑够

   例如，对于money = 4 , coins = List(1, 2)

   有3种方法（交换顺序算一种方法）

   4 = 1 + 1 + 1 + 1

      = 1 + 1 + 2

      = 2 + 2

   - 显然，当money = 0 时，结果为0

   - 当 money != 0时，设当前的答案为f(money, coins)

     那么 f(money, coins) = f(money - coins.head, conins) + f(money, coins.tail)

     即当前状态的结果，为用零钱里第一种面额的找零方案，加上不用零钱里第一种面额的找零方案，这两个结果的数目之和

     易知这个划分是互不相交且并集为完整结果集

   - 再考虑到边界条件

   代码实现如下

   ```scala
   def countChange(money: Int, coins: List[Int]): Int = {
     def count(money: Int, coins: List[Int]): Int = 
       if (coins.isEmpty) 0
       else if (money == coins.head) 1
       else if (money < coins.head) 0
       else count(money - coins.head, coins) + count(money, coins.tail)
        
     count(money, coins.sorted)
   }
   ```

   



