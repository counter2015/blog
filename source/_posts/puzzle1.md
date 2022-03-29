#  Scala Puzzle 1.Hi,There!

title:  Scala Puzzle 1.Hi,There!
date: 2018-11-18 19:19:46
tags: [scala, 读书笔记]  
categories: [puzzles]

------



官网地址：[puzzle 1](https://scalapuzzlers.com/#pzzlr-001)

### 前言

让代码做我们希望它做的事，但有时候我们本以为“已经理解”的代码表现除与我们相反的情况。

`Scala Puzzlers`就是一本这么一本有趣的书，专门收集一些例子，帮助我们认识许多反直观的雷区和陷阱。

本系列代码基于`Scala 2.11`


### Hi, There

```scala
scala> List(1, 2).map { i => i + 1 }
res1: List[Int] = List(2, 3)
```
在Scala中，只传入一个参数的方法调用，可以用花括号。以上的代码等价于
```scala
scala > List(1,2).map(i => i + 1)
res1: List[Int] = List(2, 3)
```
这能让调用方在花括号中定义函数字面量。


```scala
scala> List(1, 2).map { i => println("Hi"); i + 1 }
Hi
Hi
res2: List[Int] = List(2, 3)
```
等价于
```scala
scala> List(1,2).map(i => {println("Hi"); i + 1})
Hi
Hi
res2: List[Int] = List(2, 3)
```
但不能写成
```scala
scala> List(1,2).map(i => println("Hi"); i+1)
<console>:1: error: ')' expected but ';' found.
       List(1,2).map(i => println("Hi"); i+1)

```

同样地，我们可以使用`_`来构造匿名函数，我们可以发现以下两行代码是等效的
```scala
scala> List(1, 2).map { i => i + 1 }
res1: List[Int] = List(2, 3)

scala> List(1, 2).map { _ + 1 }
res0: List[Int] = List(2, 3)
```
那么假如我们在上面两种代码中，分别加入调试代码，会出现什么样的结果呢？

```scala
scala> List(1, 2).map { i => println("Hi"); i + 1 }
Hi
Hi
res0: List[Int] = List(2, 3)

scala> List(1, 2).map { println("Hi"); _ + 1 }
Hi
res1: List[Int] = List(2, 3)

```
为什么会出现这种情况呢？

我们知道，花括号里的语句可以被视为函数字面量，那么对于第一条语句，`map{i => println("Hi"); i+1}`中的语句，`println("Hi"); i+1`被看作一个函数字面量，其中`i+1`为这个函数字面量的返回值。为了更好地理解，我们可以看下面这个例子(构建了等效于第一条语句的命令)：
```scala
scala> val printAndAddOne = (i:Int) => {println("Hi"); i+1}
printAndAddOne: Int => Int = $$Lambda$1110/2073484941@7ccd611e

scala> List(1,2).map(printAndAddOne)
Hi
Hi
res2: List[Int] = List(2, 3)

```
以上代码，还等价于
```scala
scala> List(1, 2).map( i =>{ println("Hi"); i + 1})
Hi
Hi
res4: List[Int] = List(2, 3)
```



这是因为`{i => println("Hi"); i + 1}`被当成一个**函数字面量表达式**,`println`语句是函数体的一部分，所以每次调用函数就要执行一次。



接下来我们来看第二条语句，其实，在第二条语句中，花括号中使用了`_`，代码块被看作是两条语句，`println("Hi")`和`_+1`。整个代码块都会被执行，最后一行代码的结果将会被传入到map中，因而输出语句并不是在map语句中,以上代码等效于

```scala
scala> val printAndReturnAFunc = {println("Hi"); (_:Int)+1}
Hi
printAndReturnAFunc: Int => Int = $$Lambda$1128/2049393953@2fa47368
scala> List(1,2).map(printAndReturnAFunc)
res5: List[Int] = List(2, 3)
```
以上代码，还等效于
```scala
scala> List(1,2).map({println("Hi"); i => {i+1}})
Hi
res6: List[Int] = List(2, 3)
```

### 小结

我们通过这些例子，可以发现，匿名函数的作用范围，只延申到包含下划线的表达式，这不同于常规的匿名函数，常规的匿名函数的函数体是包含从箭头标识符`=>`一直到代码块结束的所有代码，如

```scala
scala> val regularFunc = {a : Any => println("Hi"); println(a); "2333"}
regularFunc: Any => String = <function1>

scala> regularFunc("There")
Hi
There
res0: String = 2333


```

而使用占位符语法的函数被封闭在它自己的代码块里。下面两个函数是等效的

```scala
scala> val annoymousFunc = {println("1"); println(_:Any); "23e3"}
1
annoymousFunc: String = 23e3

scala> val confinedFunc = {println("1"); {a: Any => println(a)}; "23e3"}
1
confinedFunc: String = 23e3


scala> annoymousFunc(2)
res1: Char = e

scala> confinedFunc(2)
res2: Char = e

```



> Scala代码风格偏向于简洁，但过于简洁也会导致问题，使用占位符时一定要注意它的作用范围



