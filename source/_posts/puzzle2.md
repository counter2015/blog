#  Scala Puzzle 2.UPSTAIRS downstairs

title:  Scala Puzzle 2.UPSTAIRS downstairs
date: 2019-01-06 14:29:52
tags:  [scala, 读书笔记]
categories: [puzzles]

------



官网地址：http://scalapuzzlers.com/#pzzlr-003

## 2. UPSTAIRS downstairs

Scala为我们在初始化变量时提供了遍历方法，如:

```scala
scala> var (z, q, x) = (7, 2, 1)
z: Int = 7
q: Int = 2
x: Int = 1
```

编译器为我们自动解析了`(7, 2, 1)`并赋值给`(z, q, x)`

然而，有时也会带来意想不到的结果

在[REPL](https://docs.scala-lang.org/overviews/repl/overview.html) （Read Eval Print Loop）中执行以下代码会是什么结果呢？

```scala
var MONTH = 12; var DAY = 24
var (HOUR, MINUTE, SECOND) = (12, 0, 0)
```

按照我们之前的经验，预计这两个语句都能正常运行，但是实际上却略有差别：

```scala
scala> var MONTH = 12; var DAY = 24
MONTH: Int = 12
DAY: Int = 24

scala> var (HOUR, MINUTE, SECOND) = (12, 0, 0)
<console>:11: error: not found: value HOUR
       var (HOUR, MINUTE, SECOND) = (12, 0, 0)
            ^
<console>:11: error: not found: value MINUTE
       var (HOUR, MINUTE, SECOND) = (12, 0, 0)
                  ^
<console>:11: error: not found: value SECOND
       var (HOUR, MINUTE, SECOND) = (12, 0, 0)

```

使用大写变量就会带来这个问题吗？怀着这个疑问，进行对照测试

```scala
scala> var (hour, minute, second) = (12, 0, 0)
hour: Int = 12
minute: Int = 0
second: Int = 0

```

全小写变量，没问题。

```scala
scala> var (Hour, Minute, Second) = (12, 0, 0)
<console>:11: error: not found: value Hour
       var (Hour, Minute, Second) = (12, 0, 0)
            ^
<console>:11: error: not found: value Minute
       var (Hour, Minute, Second) = (12, 0, 0)
                  ^
<console>:11: error: not found: value Second
       var (Hour, Minute, Second) = (12, 0, 0)

```

首字母大写，报错。

```scala
scala> var (hOUR, mINUTE, sECOND) = (12, 0, 0)
hOUR: Int = 12
mINUTE: Int = 0
sECOND: Int = 0

```

首字母小写，没问题。

基于以上的测试，我们可以定位问题出现的原因就在于首字母大写。

然而，为什么首字母大写会导致初始化错误呢？

## 解释

Scala允许对简单的、单个的、赋值操作`val`、`var`标识符使用大写变量名。

但是对多个的赋值操作，却不能使用大写变量名(或者更准确地说，是大写字母开头的变量名)。

这是因为多变量复制是基于模式匹配的，而在模式匹配中，大写字母有特殊含义——***静态标识符***，而静态标识符是用来匹配常量的。

这样说也许有些不好理解，那我们还是来看一个例子：

```scala
scala> val H = 12
H: Int = 12

scala> val (H, m, s) = (12, 0, 0)
m: Int = 0
s: Int = 0

scala> val (H, m, s) = (13, 0, 0)
scala.MatchError: (13,0,0) (of class scala.Tuple3)
  ... 32 elided
```

在我们定义了`H`常量的值后，第二条语句能正常运行，因为`H` match`12`。同样的道理，第三条语句就会报`MatchError`的错误。第二条语句虽能能正常运行，但是实际上并没有给`H`赋值

简单来说，大写字母开头的变量是用来匹配常量的，小写字母开头的变量是给变量赋值的。

小写变量放在反引号 **\`\`** 内，也能作为静态标识符使用，这种情况下，只能是`val`，<del>话说都val了还能算变量吗</del>

下面是一个例子。

```scala
scala> val theAnswer = 42
theAnswer: Int = 42


scala> def checkGuess(guess: Int) = guess match{
     |   case `theAnswer` => "Correct!"
     |   case _ => "Try Again."
     | }
checkGuess: (guess: Int)String

scala> checkGuess(2)
res8: String = Try Again.

scala> var theAnswer = 42
theAnswer: Int = 42

scala> def check(g: Int) = g match {
     |   case `theAnswer` => "Correct!"
     |   case _ => "Try again."
     | }
<console>:13: error: stable identifier required, but theAnswer found.
         case `theAnswer` => "Correct!"

```

## 小结

如果需要常量，用全大写字符命名,标识符为`val`，如果需要变量，用驼峰命名，如`theAnswer`,具体遵循命名规范。

**仅对常量使用大写的变量名(除了类名,如`MyClassName`)**

