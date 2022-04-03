# 记一次Scala问题的排查

title: 记一次Scala问题的排查
date: 2020-03-27 00:20:24
tags: [scala, Github]
categories: [Bugfix]

---


## 问题的发现

在看书（*Programming in Scala 3rd Edition*）时发现了这么一段代码

```scala
def lazyMap[T, U](coll: Iterable[T], f: T => U) = 
  new Iterable[U] {
    def iterator = coll.iterator map f
  }
```
这段代码看上去平平无奇，不过是把原有的`map`函数改写成了依赖`iterator`的惰性求值版本。

但是放到REPL里测试时，却出乎我的意料
```scala
Welcome to Scala 2.13.1 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181).
Type in expressions for evaluation. Or try :help.

scala> def lazyMap[T, U](coll: Iterable[T], f: T => U) = 
     |   new Iterable[U] {
     |     def iterator = coll.iterator map f
     |   }
           def iterator = coll.iterator map f
                                            ^
On line 3: error: type mismatch;
        found   : T => U
        required: U => U
```



## 排查过程

我觉得有点不对劲，这代码明明是对的，怎么就类型不匹配了呢

我的直觉觉得这个代码是没有问题的，难道这是个bug?然后我就随手开了另外一个版本的REPL测试

```scala
Welcome to Scala 2.12.6 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_111).
Type in expressions for evaluation. Or try :help.

scala> def lazyMap[T, U](coll: Iterable[T], f: T => U) = 
     |   new Iterable[U] {
     |     def iterator = coll.iterator map f
     |   }
lazyMap: [T, U](coll: Iterable[T], f: T => U)Iterable[U]
```

看起来是比较像bug了

<del>之后为了确定又去请教了万能的群友，一下子就指出问题在源码里</del>

其实这个问题是和源码有关的，以两个版本的scala源码为例

旧版本 Scala 2.11.12(不拿2.12.x版本是因为IDEA里刚好有这个版本而且看起来差不多)

```scala
// scala 2.11.12 scala-library-2.11.12-sources.jar/scala/collection/Iterable.scala
trait Iterable[+A] extends Traversable[A]
                      with GenIterable[A]
                      with GenericTraversableTemplate[A, Iterable]
                      with IterableLike[A, Iterable[A]] {
  override def companion: GenericCompanion[Iterable] = Iterable

  override def seq = this
```

新版本scala 2.13.1 这个是从Github clone下来的代码，下载了老半天

```scala
// scala 2.13.1  src/library/scala/collection/Iterable.scala

trait Iterable[+A] extends IterableOnce[A]
  with IterableOps[A, Iterable, Iterable[A]]
  with IterableFactoryDefaults[A, Iterable] {

  // The collection itself
  final def toIterable: this.type = this

  final protected def coll: this.type = this
....
```

在新版本中，可以看到`seq`貌似是被重写成了`coll`

简单测试下

```scala
Welcome to Scala 2.13.1 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181).
Type in expressions for evaluation. Or try :help.

scala> Array(1,2,3).toIterable.seq
                               ^
       warning: method seq in trait Iterable is deprecated (since 2.13.0): Iterable.seq always returns the iterable itself
res0: scala.collection.mutable.ArraySeq.ofInt = ArraySeq(1, 2, 3)

scala> Array(1,2,3).toIterable.coll
                               ^
       error: method coll in trait Iterable cannot be accessed in scala.collection.mutable.ArraySeq.ofInt
        Access to protected method coll not permitted because
        enclosing object $iw is not a subclass of
        trait Iterable in package collection where target is defined

```

果然这个在新版被废弃了，`coll`访问不了这是当然的，

因为修饰关键字为`final protected`,不可修改且只能从其子类访问。

其实已经有了初步的猜想，之前代码无法编译通过和变量名称有关。

关于这个猜想，可以在旧版本构造一个这样的例子来验证

```scala
Welcome to Scala 2.12.6 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_111).
Type in expressions for evaluation. Or try :help.

scala> def lazyMap[T, U](seq: Iterable[T], f: T => U) = 
     |   new Iterable[U] {
     |     def iterator = seq.iterator map f
     |   }
<console>:13: error: type mismatch;
 found   : T => U
 required: U => U
           def iterator = seq.iterator map f

```

OK, 完美复现

简单来说，这里出现bug的原因是，外部的`cell: Iterable[T]`被`trait`内部定义的`cell: Iterable[U]`

(继承自 `new Iterable[U]`, 这里`cell`的类型为`this.type`也就是`Iterable[U]`)

编译器在做类型推断时， 期望`lazyMap`返回一个`Iterable[U]`，

而`iterator`期望返回`Iterator[U]`类型的数据

对于迭代器里的元素类型为`U`,需要返回的是`U`类型的数据，自然可以算出`map`中函数的类型应该为`U => U`

所以就有这个报错了

```scala
<console>:13: error: type mismatch;
 found   : T => U
 required: U => U
           def iterator = seq.iterator map f
```





## 小结

这个问题有一个非常简单的解决办法，那就是重命名传入的参数，比如

```scala
def lazyMap[T, U](coll: Iterable[T], f: T => U) = {
  val _coll = coll
  new Iterable[U] {
    def iterator = _coll.iterator.map(f)
  }
}
```

但这还是有一个问题，如果这个`trait`里面包含很多很多变量，总不能一个一个去确认吧？

所以最好的办法还是编译器提供一个警告，告诉你

> 这个变量和内部作用域的某个变量重名了，你注意一点

嗯~ o(*￣▽￣*)o，混了一个Scala的[issue](https://github.com/scala/bug/issues/11921),贡献列表终于看起来终于不那么寒酸了

issue回复超快，Martin还准备对这个做优化，感觉很开心

![](https://counter2015.com/picture/scala-issue-1-1.jpg)

不负责任地猜一下优化思路，用反射从当前的上下文中获取到所有定义的`method/variable`

然后在外层判断这一层的定义域中是否和内层的定义域产生冲突。

到时候关注下[Dotty](https://github.com/lampepfl/dotty/issues/8617)里是怎么实现的好了。