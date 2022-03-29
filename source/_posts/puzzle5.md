#  Scala Puzzle 5.The Missing List

title:  Scala Puzzle 5.The Missing List
date: 2019-04-7 13:00:25
tags:  [scala, 读书笔记]
categories: [puzzles]

------

官网地址：<http://scalapuzzlers.com/#pzzlr-006>

## 问题的提出

scala的集合操作十分丰富，有时候自定义函数的作用和想象中的不一样，

例如，我想要实现一个函数，可以对集合的大小求和，对Vector("a")，List("b","c")和Array("d","e","f")的几个大小求和，输出结果是1+2+3=6

```scala
scala> sumSizes(Seq(Vector("a"), List("b", "c"), Array("d", "e", "f")))
res5: Int = 6


```

这样看起来貌似没啥问题，但是如果我们尝试如下代码

```scala
scala> def sumSizes(collections: Iterable[Iterable[_]]): Int = 
     |   collections.map(_.size).sum
sumSizes: (collections: Iterable[Iterable[_]])Int

scala> sumSizes(List(Set(1,2), List(3,4)))
res0: Int = 4

scala> sumSizes(Set(Set(1,2), List(3,4)))
res1: Int = 2

scala> sumSizes(Set(Set(1,2,4), List(3,4)))
res2: Int = 5

scala> sumSizes(Set(Set(1,2,4), List(3,4,5)))
res3: Int = 3

scala> sumSizes(List(Set(1,2,4), List(3,4,5)))
res4: Int = 6

```

我们可以发现，其中有几个是和想象中的不太一样的

- `sumSizes(Set(Set(1,2), List(3,4)))` 结果是2·
- `sumSizes(Set(Set(1,2,4), List(3,4,5)))`结果是3

但是对比下面几个

- `sumSizes(List(Set(1,2), List(3,4)))`结果是4
- `sumSizes(List(Set(1,2,4), List(3,4,5)))`结果是6

我们发现，区别只在于把`Set`换成了`List`,再仔细看看之前定义的函数

```scala
def sumSizes(collections: Iterable[Iterable[_]]): Int = 
  collections.map(_.size).sum
```

以上函数等价于

```scala
def SumSizes(collections: Iterable[Iterable[Any]]): Int = 
  collections.map(collection => collection.size).sum
```

函数名为`sumSizes` 函数接受的输入类型为 `Iterable[Iterable[Any]]`,不严谨地说叫做集合的集合。

函数的输出结果为整型数，含义为`Iterable[Iterable[Any]]`中每一个`Iterable[Any]`的`size`，然后对其求和。

## 解释

让我们来（非形式化地）模拟一遍函数的执行过程(以`sumSizes(List(Set(1,2), List(3,4)))`为例)

1. sumSizes(List(Set(1,2), List(3,4)))
2. collection.map(List(Set(1,2), List(3,4)) => List(Set(1,2).size, List(3,4).size)),sum
3. List(2,2).sum
4. 结果为4

同样地，对`sumSizes(Set(Set(1,2), List(3,4)))`进行类似的计算

1. sumSizes(Set(Set(1,2), List(3,4)))
2. collection.map(Set(Set(1,2), List(3,4)) => Set(Set(1,2).size, List(3,4).size)),sum
3. Set(2,2).sum
4. Set(2).sum
5. 结果为2

这样原因就很明了了，这是由于map操作`map(T => f(T)): CollectionType[B]` 其中 B为f(T)计算结果的类型，

在这个例子中可以写作`map(T => T.size): collectonType[Int]`

对于Set[T]返回的结果也是Set[T],

而Set的特性保证相同的元素在集合中只有一个，而计算的中间结果正好有重复结果结果Set(2,2)就成了Set(2)

求和的结果正好就是2了。

## 讨论

要想避免这种意外结果，在不需要保留输入类型的情况下，

可以手动调整中间类型（不一定非要是`toSeq`，例如`toList`也是同样的道理）

```scala
scala> def sumSizes(collections: Iterable[Iterable[_]]): Int = 
     |   collections.toSeq.map(_.size).sum
sumSizes: (collections: Iterable[Iterable[_]])Int

scala> sumSizes(Set(Set(1,2), List(3,4)))
res10: Int = 4


```

还有一种可选的方法是使用fold来实现累加功能，消除来自于集合外部的访问 可能对中间结果造成的影响

（直接sum的话，如果集合为Set类型，会消除中间结果中的重复元素，造成不符合预期的结果）

```scala
scala> def sumSizes(collections: Iterable[Iterable[_]]): Int =
     |   collections.foldLeft(0) {
     |     (sum, collection) => sum + collection.size
     |   }
sumSizes: (collections: Iterable[Iterable[_]])Int

scala> sumSizes(Set(Set(1,2), List(3,4)))
res0: Int = 4

```

foldLeft的函数签名(function signature)如下

`def foldLeft[B](z: B)(f: (B, A) => B): B`

执行过程类似于

对于

`A = [a0, a1, ... , an]`

`A.foldLeft(z)(f)` = `f(...f(f(z, a0), a1) ...), an)`

为什么不用`fold,foldRight`是因为考虑到简便和操作符的结合顺序的问题，

更多关于`fold`的说明，可以参考[这里](<https://commitlogs.com/2016/09/10/scala-fold-foldleft-and-foldright/>)

同样地，我们来模拟下这个函数的运行过程

1. sumSizes(Set(Set(1,2), List(3,4)))

2. Set(Set(1,2), List(3,4)).foldLeft(0) { 

     (sum, collection) => sum + collection.size

   }

3. foldLeft Steps

   - sum = 0, collection = Set(1,2),  sum = sum + collection.size = 0 + 2 = 2
   - sum = 2, collection = List(3,4), sum = sum + collection.size = 2 + 2 = 4

4. 结果为4

## 小结

对集合进行操作时，需要特别注意其输入的类型。