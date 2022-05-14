# FPDIS 2.Lazy Evaluation

title: FPDIS 2.Lazy Evaluation
date: 2020-08-27 00:47:30
tags: [Scala, 读书笔记, Coursera] 
categories: [技术]

------




## Structural Induction on Trees

Martin开篇耿直的说这对于在线课程是可选的(暗示可以跳过)

但是马上又补充了一句说如果你是EPFL的学生，最好别跳，因为考试可能会考

[之前的课程](http://counter2015.com/2019/09/04/fp-in-scala3/#%E7%BC%96%E7%A8%8B%E4%BD%9C%E4%B8%9A-fucntional-sets)中，我们写了一个function set

```scala
abstract class Intset {
  def incl(x: Int): IntSet
  def contains(x: Int): Boolean
}

object Empty extends IntSet {
  def incl(x: Int): IntSet = NonEmpty(x, Empty, Empty)
  def contains(x: Int): Boolean = false
}

case class NonEmpty(elem: Int, left: IntSet, right: IntSet) extends IntSet {
  def incl(x: Int): IntSet = {
    if (x < elem) NonEmpty(elem, left incl x, right)
    else if (x > elem) NonEmpty(elem, left, right incl x)
    else this
  }
  
  def contains(x: Int): Boolean = 
    if (x < elem) left contains x
    else if (x > elem) right contains x
    else true
}
```

如果我们想证明对于某个树T，满足性质P，即P(T)

- 需要先证明对于所有叶子节点l，有P(l)
- 再需要证明对于所有的非叶子节点t, 对于它的所有子树集合s1,...,sn，都满足P(s1) ^ ... ^ P(sn)

对于这段代码，需要证明它的正确性

那么，什么是代码的**正确性**呢

这里指的是代码按照我们预期的方式进行工作，比如，我们可以定义如下规则

- Empty contains x = false
- (s incl x) contains x = true
- (s incl x) contains y = s contians y

对于这段代码，该如何证明呢？

> 即得易见平凡，仿照上例显然。留作习题答案略，读者自证不难。
> 反之亦然同理，推论自然成立，略去过程QED，由上可知证毕



`Empty contains x = false` 这条最简单，看定义易得

```scala
object Empty extends IntSet {
  def incl(x: Int): IntSet = NonEmpty(x, Empty, Empty)
  def contains(x: Int): Boolean = false
}
```

`(s incl x) contains x = true`

对于Empty的情况

```
(Empty incl x) contains x
= NonEmpty(x, Empty, Empty) contians x //由Empty.incl定义可知
= true //由NonEmpty.contains的定义可知
```

对于NonEmpty(x, l, r)

```
(NonEmpty(x, l, r) incl x) contains x
= NonEmpty(x, l, r) contains x
= true
```

对于NonEmpty(z, l, r), z < x

```
(NonEmpty(z, l, r) incl x) contains x
= NonEmpty(z, l, r incl x) contains x
= (r incl x) cotains x
= true // 由NonEmpty(x, l, r)的推论可得
```

对于NonEmpty(z, l, r), z > x 同理可得

`(s incl x) contains y = s contians y`

第三种情况留作习题



## Streams, or LazyList ?

从 Scala 2.13 版本开始，`Streams`被废弃，推荐使用`LazyList`，所以下面用到的例子都是`LazyList`

```scala
val xs = LazyList.cons(1, LazyList.cons(2, LazyList.empty))
val ys = LazyList(1,2,3)
```

和`List`做个简单对比

```scala
def lazyListRange(lo: Int, hi: Int): LazyList[Int] = 
  if (lo >= hi) LazyList.empty else LazyList.cons(lo, streamRange(lo+1, hi))

def listRange(lo: Int, hi: Int): List[Int] = 
  if (lo >= hi) List.empty else lo :: listRange(lo+1, hi)
```

两者的结构基本相同，不同点在于元素是否立刻计算

嗯，还可以用特殊的符号`#::`代替`LazyList.cons`

这里的延迟计算实际上使用传名参数来实现的(call by name)

下面是一个简单的函数签名示例(仅供参考，并不能通过编译)

```scala
trait LazyList[+A] extends Seq[A] {
  def isEmpty: Boolean
  def head: A
  def tail: LazyList[A]
  ...
}

object LazyList {
  def cons[T](hd: T, tl: => LazyList[T]) = new LazyList[T] {
    override def isEmpty = false
    override def head = hd
    override def tail = tl
  }
  
  val empty = new LazyList[Nothing] {
    override def isEmpty = true
    override def head = throw new NoSuchElementException("empty.head")
    override def tail = throw new NoSuchElementException("empty.tail")
  }
}
```





本节的习题

```scala
def lazyListRange(lo: Int, hi: Int): LazyList[Int] = {
  print(lo+" ")
  if (lo >= hi) LazyList.empty else LazyList.cons(lo, streamRange(lo+1, hi))
}

lazyListRange(1, 10).take(3) 输出什么
```



- [ ] Nothing
- [ ] 1
- [x] 1 2 3
- [ ] 1 2 3 4
- [ ] 1 2 3 4 5 6 7 8 9



## Lazy Evaluation

什么是`Lazy`？

> Do things as late as possible and never do them twice.

这一点其实受到 Haskell 相当大的影响——这个语言默认就是 Lazy 计算的

为什么 Scala 不这样做呢？

Martin给出的解释：

**Scala 允许可变的变量和副作用，这个和Lazy一起搞容易出玄学错误**

e.g. 从 StackOverflow 找的一个典型例子

```scala
scala> def square(a: =>Int) = a * a
def square(a: => Int): Int

scala> square({println("calculating");5})
calculating
calculating
val res0: Int = 25
```

由于入参 `a` 是 Call by name 的形式，所以是被延迟计算的，这会导致在`a * a`的步骤中被调用了两次`println`的副作用——这恰恰是我们不希望发生，却又很容易被忽视的一个问题

所以 Scala 被设计成默认立即计算，但是也可以支持延迟计算



本节习题：

```scala
def expr = {
  val x = { print("x"); 1}
  lazy val y = { print("y"); 2}
  def z = { print("z"); 3}
  z + y + x + z + y + x
}
expr
```

执行`expr`的副作用导致输出结果是什么

- [ ] zyxzyx
- [x] xzyz
- [ ] xyzz
- [ ] zyzz
- [ ] something else 



## Computing with Infinite Sequences

本节简单介绍了下LazyList的应用

- 筛法求素数
- 牛顿开方

习题 

``` scala
val N = 3
def from(n: Int): LazyList[Int] = n #:: from(n + 1)
val xs = from(1) map (_ * N)
val ys = from(1) filter (_ % N == 0)
```

上面两个表达式哪个生成速度较快？

显然答案是上面那个表达式，因为`map`操作不需要额外生成新的元素，

而`filter`操作则是每隔`N`个元素要判断`N-1`次

## Case Study: the Water Pouring Problem

本节探讨的问题是[倒水问题](https://en.wikipedia.org/wiki/Water_pouring_puzzle)

倒水问题的简单版本，就是已知有一个水龙头，和一个足够大的水槽，有两个不同容量，没有刻度的水杯。

现在允许做的操作有

- 把水导入水槽
- 从水龙头接水把杯子装满水
- 把一个水杯的水倒入另外一个水杯，知道某个水杯为空或者倒满

现在问，为了量出若干容量的水，所需要的步骤是什么



这个问题讲的很好，Martin一步步抽象问题，解释每一步的意图。

这里就不具体讲代码了，可以直接看课程



## 编程作业Bloxorz

这个编程作业，是对一个游戏的简化版本自动求解，游戏可以在[这里](https://www.miniclip.com/games/bloxorz/en/)玩到



### Game Setup

位置的定义如下

```scala
case class Pos(row: Int, col: Int)
```

- 行坐标确定垂直方向的位置
- 列坐标确定水平方向的位置
- 坐标原点在左上角，坐标值从上到下，从左到右依次增加

这里用的是坐标系

```
  0 1 2 3   <- col axis
0 o o o o
1 o o o o
2 o # o o    # is at position Pos(2, 1)
3 o o o o

^
|

row axis
```



我们这样定义地面(Terrain)

```scala
type Terrain = Pos => Boolean
```

地面可以使用`StringParserTerrain.scala`里的方法快速创建



我们需要先实现这个文件里的方法

```scala
def terrainFunction(levelVector: Vector[Vector[Char]]): Pos => Boolean = ???
def findChar(c: Char, levelVector: Vector[Vector[Char]]): Pos = ???
```

具体的要求，在对应的 Scaladoc 里能找到

一个简单的实现如下

```scala
  def terrainFunction(levelVector: Vector[Vector[Char]]): Pos => Boolean = {
    val rows = levelVector.length
    val cols = levelVector.head.length
    def valid(r: Int, c: Int): Boolean = {
      r >= 0 && c >= 0 && r < rows && c < cols
    }
    (p: Pos) => valid(p.row, p.col) && levelVector(p.row)(p.col) != '-'
  }

def findChar(c: Char, levelVector: Vector[Vector[Char]]): Pos = {
  val index = levelVector.flatten.indexWhere(p => p == c)
  val cols = levelVector.head.length
  Pos(index / cols, index % cols)
}
```



对场景的函数做好定义后，我们还需要定义好砖块

砖块可以被视为 2\*1\*1 的立方体

需要用两个字段来描述

```scala
case class Block(b1: Pos, b2: Pos)
```

砖块可以往平面上的四个方向滚动

需要实现这么几个函数

```scala
// 砖块是否处于直立状态
def isStanding: Boolean = ???

// 砖块是否处于地面上
def isLegal: Boolean = ???

// 砖块的初始状态
def startBlock: Block = ???
```

下面是一个简单的实现

```scala
def isStanding: Boolean = b1 == b2
def isLegal: Boolean = terrain(b1) && terrain(b2)
def startBlock: Block = Block(startPos, startPos)
```



如上所言，我们需要让砖块能在平面上的四个方向滚动,滚动时需要判定新位置是否合法

```scala
    /**
     * Returns the list of blocks that can be obtained by moving
     * the current block, together with the corresponding move.
     */
    def neighbors: List[(Block, Move)] = ???

    /**
     * Returns the list of positions reachable from the current block
     * which are inside the terrain.
     */
    def legalNeighbors: List[(Block, Move)] = ???

```



简单实现如下

```scala
    def neighbors: List[(Block, Move)] = {
      List(
        (left, Left),
        (right, Right),
        (up, Up),
        (down, Down)
      )
    }

    def legalNeighbors: List[(Block, Move)] = {
      neighbors.filter(x => terrain(x._1.b1) && terrain(x._1.b2))
    }
```



### Solving the Game


我们把前期的准备工作做完了，接下来就是具体的求解 `Solver.scala` 实现了

我们可以用`LazyList[Block]`来表示具体的解，但是从倒水问题的经验来看，还是很有必要保存之前达到过的状态的，避免重复走到之前走过的位置。



因此，我们用`LazyList[(Block, List[Move])]`来表示解答

其中，第二部分`List[Move]`表示之前的移动路径

最后一个移动路径，是`List[Move]`的首元素 （由于List在头部插入效率远高于尾部插入）



判断是否到达终点的代码很简单

```scala
  def done(b: Block): Boolean = {
    b.b1 == b.b2 && b.b1 == goal
  }
```



然后是实现获取下一个状态, 并排除掉已经经过的状态

```scala
  def neighborsWithHistory(b: Block, history: List[Move]): LazyList[(Block, List[Move])] = {
    LazyList(b.legalNeighbors: _*).map{case (block, move) => (block, move :: history)}
  }

  def newNeighborsOnly(neighbors: LazyList[(Block, List[Move])],
                       explored: Set[Block]): LazyList[(Block, List[Move])] = {
    neighbors.filter(b => !explored.contains(b._1))
  }
```



### Finding Solutions

现在问题的关键在如何求解

现在需要构造`from`函数，这个函数会更加初始砖块的位置和已经遍历过的位置，求出所有可能的解空间集合

```scala
def from(initial: LazyList[(Block, List[Move])],
         explored: Set[Block]): LazyList[(Block, List[Move])] = ???
```



这里可以参照倒水问题的写法

```scala
  def from(initial: LazyList[(Block, List[Move])],
           explored: Set[Block]): LazyList[(Block, List[Move])] = {
    if (initial.isEmpty) LazyList.empty
    else {
      val more = for {
        (block, history) <- initial
        neighbors = neighborsWithHistory(block, history)
        newNeighbors = newNeighborsOnly(neighbors, explored)
        neighbor <- newNeighbors
      } yield neighbor
      initial #::: from(more, explored ++ more.map(_._1))
    }
  }
```

详细讲解下这段代码的意图

- 首先处理传参为空的情况
- 对于非空的情况，获取到当前状态的下一个状态
- 过滤掉之前抵达过的砖块状态

把这些结果保存下来，作为下一步可以到达的解空间，同时维护已经抵达过的状态数

剩下的几个函数就是水到渠成的了

```scala
  lazy val pathsFromStart: LazyList[(Block, List[Move])] = {
    from(LazyList((startBlock, Nil)), Set[Block]())
  }

  lazy val pathsToGoal: LazyList[(Block, List[Move])] = {
    pathsFromStart.filter(x => done(x._1))
  }

  lazy val solution: List[Move] = pathsToGoal.headOption.map(_._2).getOrElse(Nil).reverse
```

