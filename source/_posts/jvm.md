# StackoverFlow and JVM

title: StackoverFlow and JVM
date: 2019-02-24 15:05:24
tags: jvm
categories: [技术]

---



在学习scala的过程中注意到了这么一段示例代码

```scala
def factorial(x: BigInt): BigInt = if (x == 0) 1 else x * factorial(x - 1)
```

这是一个计算阶乘的代码，这段代码是递归的方式写的，如果x的值足够大，就会栈溢出，那么到底栈的深度到底是多少呢？

为此，写了如下代码来计算栈的深度

```scala
def stackDepth():Int = 
      try {
        1 + stackDepth()
      } 
      catch {
        case e: StackOverflowError => 0
      }
```

以下是在REPL中运行的结果

```scala
scala> def stackDepth():Int = try {1 + stackDepth()} catch {case e: StackOverflowError => 0}
stackDepth: ()Int

scala> stackDepth
res1: Int = 21196

scala> stackDepth
res2: Int = 58829

scala> stackDepth
res3: Int = 58829

scala> stackDepth
res4: Int = 58829

scala> def stackDepth():Int = try {1 + stackDepth()} catch {case e: StackOverflowError => 0}
stackDepth: ()Int

scala> stackDepth
res5: Int = 9834

scala> stackDepth
res6: Int = 23531

scala> stackDepth
res7: Int = 58829

scala> stackDepth
res8: Int = 58829

```



我们可以观察到一些有意思的现象

- 同一个函数，多次运行时会在不同的位置抛出异常
- 随着运行次数的增加，结果会趋于稳定，这里最后稳定的结果是`58829`
- 如果重新定义函数，栈深的测量结果会变小，但随着运行次数的增加，最后会和之前的结果一致



这是为什么呢？

其实仔细观察，在启动REPL的时候scala会打印如下信息

```shell
$ scala
Welcome to Scala 2.11.12 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181).
Type in expressions for evaluation. Or try :help.

scala> 

```

会简单显示scala版本,java版本和使用的jvm（Java Virtual Machine）名称，而答案正是和jvm有关。
>  Hotspot VM的热点代码探测能力可以通过执行计数器找出最有编译价值的代码，然后通知JIT编译器以方法为单位进行编译。如果一个方法被频繁调用，或方法中的有效循环次数过多，将会分别触发标准编译和OSR（栈上替换）编译动作。                                                                                     《深入理解java虚拟机》

## JIT

>  Just-In-Time（JIT）编译器是Java Runtime Environment的一个组件，可在运行时提高Java应用程序的性能。Java程序由类组成，这些类包含可由JVM在许多不同计算机体系结构上解释的平台中性字节码。在运行时，JVM加载类文件，确定每个字节码的语义，并执行适当的计算。解释期间额外的处理器和内存使用意味着Java应用程序的执行速度比本机应用程序慢。JIT编译器通过在运行时将字节码编译为本机机器代码来帮助提高Java程序的性能。                                                     [Understanding JIT compiler (just-in-time compiler)](https://aboullaite.me/understanding-jit-compiler-just-in-time-compiler/)

  

这里的误差就是后台JIT编译造成的，如果我们尝试关闭JIT,编译速度会变慢，计算的结果也不会发生改变。

```shell
$ scala -Djava.compiler=NONE
Welcome to Scala 2.12.6 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_141).
Type in expressions for evaluation. Or try :help.

scala> def stackDepth():Int = try {1 + stackDepth()} catch {case e: StackOverflowError => 0}
stackDepth: ()Int

scala> stackDepth
res0: Int = 9092

scala> stackDepth
res1: Int = 9092

scala> stackDepth
res2: Int = 9092

```



对之前的JIT优化过程的一个简要说明如下：

1. `stackDepth`在解释器中运行
2. 多次调用后，`HotSpot`检测到这段代码，触发编译操作和栈上替换
3. 在编译期间，我们的程序还继续跑着
4. 编译完成后，`stackDepth`的函数入口被替换成优化的代码
5. JIT的优化步骤可能会有多次

因而，最开始的时候，只有解释的代码运行，后来是解释和编译的代码一起运行，当完全优化完后，就只有编译好的代码在跑着了。也就是说，栈的大小是没有变化的，变化的是我们这个程序的代码大小——逐渐优化导致每次执行的结果都不同。



## 参考资料

1. https://stackoverflow.com/questions/54759815/scala-code-in-recursive-calculation-of-factorial-function-sometimes-throw-stacko
2. https://stackoverflow.com/questions/27043922/why-is-the-max-recursion-depth-i-can-reach-non-deterministic
3. https://stackoverflow.com/questions/35517934/why-does-the-count-of-calls-of-a-recursive-method-causing-a-stackoverflowerror-v
4. programming in scala 3rd
5. 深入理解java虚拟机-jvm高级特性于最佳实践 第二版



