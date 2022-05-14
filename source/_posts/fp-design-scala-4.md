# FPDIS 4. Functions and State

title: FPDIS 4.Functions and State
date: 2020-09-10 23:43:30
tags: [Scala, 读书笔记, Coursera] 
categories: [技术]
mathjax: true

------



这周的课程没有作业，全是视频

## Functions and State

首先我们需要回忆函数求值的过程，举一个例子

```scala
def iterate(n: Int, f: Int => Int, x: Int) = 
  if (n == 0) x else iterate(n-1, f, f(x))

def square(x: Int) = x * x
```

在这个定义下，求`iterate(1, square, 3)`

```text
iterate(1, square, 3)
= if (1 == 0) x else iterate(1-1, square, square(3))
= iterate(0, square, square(3))
= iterate(0, square, 3 * 3)
= iterate(0, square, 9)
= if (0 == 0) 9 else iterate(0-1, square, square(9))
= 9
```

在这个变换过程中，可以随时选择任意一个路径化简，而不影响结果

对于`if (1 == 0) x else iterate(1-1, square, square(3))`

- `iterate(1-1, square, square(3))`
- `if (1 == 0) x else iterate(1-1, square, 3 * 3)`

以上两者是等价的。

因为最后的结果总是会交汇的。

<del>这个规律有个难记的名字叫 Church-Rosser Theorem</del>

上面的情况适用于没有状态转换的情况，如果我们引入的状态/副作用呢？

以存款取款为例

```scala
class BankAccount {
  private var balance = 0
  def deposit(amount: Int): Unit = {
    if (amount > 0) balance = balance + amount
  }
  def withdraw(amount: Int): Int = {
    if (0 < amount && amount <= balance) {
      balance = balance - amount
      balance
    } else throw new Error("insufficient funds")
  }
}
```

在REPL里测试下

```scala
scala> val acc = new BankAccount
val acc: BankAccount = BankAccount@6e090aaa

scala> acc deposit 50

scala> acc withdraw 20
val res5: Int = 30

scala> acc withdraw 20
val res6: Int = 10

scala> acc withdraw 20
java.lang.Error: insufficient funds
  at BankAccount.withdraw(<pastie>:10)
  ... 32 elided

```

对于有状态类，我们对它进行同样的操作，结果会不同



## Identity and Change

如果用`E`来代替抽象的表达式，如果

`val x = E; val y = E`并且`x`等于`y`

那么我们可以把上述表达式改写成`val x = E; val y = x`

这样的性质叫做[引用透明(referential transparency)](https://en.wikipedia.org/wiki/Referential_transparency)

这有什么好处呢？

- 可以帮助证明程序的正确性(参照之前的数学归纳法证明)
- 可以便于重构和修改代码(由于不会修改到可变的状态)
- 可以通过记忆化，公共子表达式消除、惰性求值、或并行化来优化代码

但是现实生活中很多时候不满足这个性质

以之前的账户来举例

```scala
val 你 = new BankAccount
val 我 = new BankAccount
```

你和我都开了一个账户，我们的账户显然不一样。

为什么不一样呢？因为`BankAccount`中存放了可变的值`余额`



## Loops

首先看一个计算乘方的例子

```scala
def power(x: Double, exp: Int): Double = {
  var r = 1.0
  var i = exp
  while (i > 0) {
    r = r * x
    i = i - 1
  }
  r
}
```

如果是习惯了命令式写法的选手，会觉得这个代码真是莫名其妙

为什么我不能直接对传入的参数修改，还得多此一举地声明两个新的变量`r`和`i`

还记得上一个小节提到地`引用透明`吗？其实这里的`while`循环不是必要的，我们可以用函数来消除其中不必要的状态量`r`和`i`

```scala
def WHILE(condition: => Boolean)(command: => Unit): Unit = {
  if (condition) {
    command 
    WHILE(condition)(command)
  } else ()
}
```

啊这，为什么写成这个鬼样子，原版的`while`不是简单多了？改成递归不怕爆栈吗？说好的参数是不能修改的，不就成死循环了？

不要急，下面会逐一解释。

我们可以看到，这个函数是柯里化的，第一个参数和第二个参数都是`call by name`

之所以设计成`call by name`，是为了能在每次调用时，都能感知到状态的变化，而不是传入时立即计算出结果，

之后就不变了，所以不会死循环。

另一方面，函数的返回值和自身的签名相同，或者是一个基本类型(对，Unit也是基本类型)

而这恰好满足了尾递归优化的条件，编译器会帮助我们将其展开成迭代的方式，这样只会用到常量大小的栈空间，从而避免的爆栈(StackOverflow)

```scala
scala> def WHILE(condition: => Boolean)(command: => Unit): Unit = {
     |   if (condition) {
     |     command 
     |     WHILE(condition)(command)
     |   } else ()
     | }
def WHILE(condition: => Boolean)(command: => Unit): Unit

scala> val i = 5
val i: Int = 5

scala> var i = 5
var i: Int = 5

scala> var sum = 0
var sum: Int = 0

scala> while (i > 0)({sum += i; i-=1})

scala> sum
val res1: Int = 15
```



## Extend Example: Discrete Event Simlation

本周课程的剩余时间，主要讲解如何实现一个模拟电路

由于个人的兴趣（模电数电的阴影），下面的内容就跳过了（可能之后会补上）

有兴趣的可以去了解下，如何用[chisel3](https://github.com/freechipsproject/chisel3)实现5级指令流水线
