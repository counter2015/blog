#  JetBrains Quest 2020 complete answer

title:  JetBrains Quest 2020 complete answer
date: 2020-03-11 13:00:25
tags:  [JetBrains, 猜谜]
categories: [其他]
description:  咕果是个好引擎

------



**如果你幸运地在 北京时间 2020-03-15 19:00 前完成所有谜题，你将获得JetBrains 全家桶3 * 2 个月的免费使用(续订)权**(可以使用在同一个邮箱上)**以及 八折优惠券**

UPD：

- 2020-03-19 JetBrains已经把链接的各个入口取消了，这篇文章的步骤已经无法复现了，只能留作纪念。

- 2020-04-01 JetBrains通过邮件发送了[官方题解](https://docs.google.com/document/d/1FfgW0-EryToXFjBOc9xS_bkXj2KbnVVSNnn8rSlFnIk)

偶然刷推特的时候发现了一个有趣的[活动](https://twitter.com/jetbrains/status/1236986174075482113)，做完后马上就蹭热点水了一篇文章。


![](https://counter2015.com/picture/jb-quest-2020-1.jpg)

> JetBrains Quest begins… 
>
> [#JetBrainsQuest](https://twitter.com/hashtag/JetBrainsQuest?src=hashtag_click) 48 61 76 65 20 79 6f 75 20 73 65 65 6e 20 74 68 65 20 73 6f 75 72 63 65 20 63 6f 64 65 20 6f 66 20 74 68 65 20 4a 65 74 42 72 61 69 6e 73 20 77 65 62 73 69 74 65 3f
>
> 8:04 PM · Mar 9, 2020·[Twitter Web App](https://help.twitter.com/using-twitter/how-to-tweet#source-labels)



## Part1 
### ASCII

这一串数字是个什么意思呢？

`48 61 76 65 20 79 6f 75 20 73 65 65 6e 20 74 68 65 20 73 6f 75 72 63 65 20 63 6f 64 65 20 6f 66 20 74 68 65 20 4a 65 74 42 72 61 69 6e 73 20 77 65 62 73 69 74 65 3f`

一眼看上去有点像内存里的十六进制数

那就先把它转换下呗

```scala
scala> val line = "48 61 76 65 20 79 6f 75 20 73 65 65 6e 20 74 68 65 20 73 6f 75 72 63 65 20 63 6f 64 65 20 6f 66 20 74 68 65 20 4a 65 74 42 72 61 69 6e 73 20 77 65 62 73 69 74 65 3f"
line: String = 48 61 76 65 20 79 6f 75 20 73 65 65 6e 20 74 68 65 20 73 6f 75 72 63 65 20 63 6f 64 65 20 6f 66 20 74 68 65 20 4a 65 74 42 72 61 69 6e 73 20 77 65 62 73 69 74 65 3f

scala> line.split(" ").map(Integer.parseInt(_, 16))
res0: Array[Int] = Array(72, 97, 118, 101, 32, 121, 111, 117, 32, 115, 101, 101, 110, 32, 116, 104, 101, 32, 115, 111, 117, 114, 99, 101, 32, 99, 111, 100, 101, 32, 111, 102, 32, 116, 104, 101, 32, 74, 101, 116, 66, 114, 97, 105, 110, 115, 32, 119, 101, 98, 115, 105, 116, 101, 63)

scala> res0.sorted
res1: Array[Int] = Array(32, 32, 32, 32, 32, 32, 32, 32, 32, 63, 66, 72, 74, 97, 97, 98, 99, 99, 100, 101, 101, 101, 101, 101, 101, 101, 101, 101, 101, 102, 104, 104, 105, 105, 110, 110, 111, 111, 111, 111, 114, 114, 115, 115, 115, 115, 116, 116, 116, 116, 117, 117, 118, 119, 121)
```

观察下对应的结果，最小值是32，多次出现，最大值是121

这让人联想到[ASCII码](https://en.wikipedia.org/wiki/ASCII)

在ASCII码中，32正是代表空格的位置，那么对应的也就说的通了，这是一个由空格分隔的英文短句

```scala
scala> res0.map(_.toChar).mkString
res2: String = Have you seen the source code of the JetBrains website?
```

我们得到了原文，这一步完成，下面是下一步。

###  WebSite Code

根据之前的提示，来到[官网](https://www.jetbrains.com/)

这里我的第一反应是F12看控制台，然后发现了这个。

![](https://counter2015.com/picture/jb-quest-2020-2.jpg)
接下来是我的迷惑操作，把url里面无法解析的神秘代码尝试输入到JetBrains的激活码中，结果当然是无效的。

发现自己的思路有问题，这时候就需要求助搜索引擎了

*let me google [that](https://www.google.com/search?q=JetBrainsQuest) for you*

![](https://counter2015.com/picture/jb-quest-2020-3.jpg)

如果你错误地尝试的是某度搜索，你会搜到[这个](https://www.cnblogs.com/amiaojiang/p/12459320.html)，博主吐槽


> 首先取了动态中的"JetBrains Quest"百度，结果答案为0，果然百度太垃圾了。 立刻谷歌之
> 

吐槽下博主一篇文章发了三个地方(博客园，[独立域名](https://miion.me/2020/03/10/%E5%A6%82%E4%BD%95%E7%99%BD%E5%AB%963%E4%B8%AA%E6%9C%88%E7%9A%84JetBrains%E5%85%A8%E5%AE%B6%E6%A1%B6/)，[SegmentFault](https://segmentfault.com/a/1190000021974701)),

要不是ID用的同一个我还以为是被别人抄过去了。

{% spoiler 其实这个搜索结果已经暴露了很多信息 %}

第四条就是某歪果仁写的[题解](https://mihajlonesic.gitlab.io/archive/jetbrains-quest/)

答案已经确保基本有了，接下来让我们尝试尽量少看参考答案的情况下进行独立解答

<del>凭本事搜到的答案，怎么能说是抄呢</del>

通过题解，纠正了之前的思路，原来是需要**查看源代码**，之前的尝试有点偏了

要想看到网页源代码还是很简单的，以Chrome浏览器80.0.3987.132为例。
到官网后右键单击查看源代码，或者在地址栏直接输入`view-source:https://www.jetbrains.com/`即可。

定位到可疑部分如下


```text
<!--
      O
{o)xxx|===============-
      O

Welcome to the JetBrains Quest.

What awaits ahead is a series of challenges. Each one will require a little initiative, a little thinking, and a whole lot of JetBrains to get to the end. Cheating is allowed and in some places encouraged. You have until the 15th of March at 12:00 CET to finish all the quests.
Getting to the end of each quest will earn you a reward.
Let the quest commence!

JetBrains has a lot of products, but there is one that looks like a joke on our Products page, you should start there... (hint: use Chrome Incognito mode)
It’s dangerous to go alone take this key: Good luck! == Jrrg#oxfn$

                 O
-===============|xxx(o}
                 O
-->
```

详细信息就不多说了，提取下里面的关键信息

- 欢迎你找到了谜题的入口
- 在 2020-03-15 12:00 CET 时间前完成一系列谜题，可以获得奖品
- 重点： 问题，JetBrains有很多产品，在产品列表中有一个像玩笑一样的东西，找出来，从这开始下一步
- 提示： 使用Chrome 无痕浏览模式(JetBrains: 谷歌结算下广告费 Google: 我帮你钦定了Kotlin来着)
- 独自上路过于危险，带上这个吧`Good luck! == Jrrg#oxfn$` 
- 所以最后画的两把🗡 是在玩村好剑的梗？

如此，我们进入到下一个谜题

### Joke Product
上一个谜题得知的信息，希望我们找到某个专门为谜题准备的恶搞产品。
从官网的导航栏开始，在 Tools 下拉栏里点击 Find your tool 进入[产品页](https://www.jetbrains.com/products.html)

![](https://counter2015.com/picture/jb-quest-2020-4.jpg)

你的画风与众不同，就是你了

点开查看详细信息

![](https://counter2015.com/picture/jb-quest-2020-5.jpg)

这个谜题需要我们计算出 500 到 5000 内有多少个质数，计算结果是一个三位数，把它填入到
` https://jb.gg/### ` 以替换`###`进入下一步

写个脚本算一下

```scala
scala> val primes: LazyList[Int] = 
  2 #:: LazyList.from(3,2).filter(
    (n: Int) => primes.takeWhile(p => p*p <= n)
      .forall(n % _ != 0))
primes: LazyList[Int] = LazyList(<not computed>)

scala> (primes.takeWhile(_ <= 5000).toList.length -
     | primes.takeWhile(_ < 500).toList.length)
res5: Int = 574
```

接下来访问` https://jb.gg/574`

发现跳转到如下页面

![](https://counter2015.com/picture/jb-quest-2020-6.jpg)

这题我会，YT是YouTrack的意思，用来提issue的，我去年提的[issue]( https://youtrack.jetbrains.com/issue/SCL-16251)现在都还没在正式版修复呢

进入到对应issue的链接： https://youtrack.jetbrains.com/issue/MPS-31816
```text
JetBrains Quest

“The key is to think back to the beginning.” – The JetBrains Quest team

Qlfh$#Li#|rx#duh#uhdglqj#wklv#|rx#pxvw#kdyh#zrunhg#rxw#krz#wr#ghfu|sw#lw1#Wklv#lv#rxu#lvvxh#wudfnhu#ghvljqhg#iru#djloh#whdpv1#Lw#lv#iuhh#iru#xs#wr#6#xvhuv#lq#Forxg#dqg#iru#43#xvhuv#lq#Vwdqgdorqh/#vr#li#|rx#zdqw#wr#jlyh#lw#d#jr#lq#|rxu#whdp#wkhq#zh#wrwdoo|#uhfrpphqg#lw1#|rx#kdyh#ilqlvkhg#wkh#iluvw#Txhvw/#qrz#lw“v#wlph#wr#uhghhp#|rxu#iluvw#sul}h1#Wkh#frgh#iru#wkh#iluvw#txhvw#lv#‟WkhGulyhWrGhyhors†1#Jr#wr#wkh#Txhvw#Sdjh#dqg#xvh#wkh#frgh#wr#fodlp#|rxu#sul}h1#kwwsv=22zzz1mhweudlqv1frp2surpr2txhvw2



上面是旧的，新的为
Qlfh$#Li#|rx#duh#uhdglqj#wklv#|rx#pxvw#kdyh#zrunhg#rxw#krz#wr#ghfu|sw#lw1#Wklv#lv#rxu#lvvxh#wudfnhu#ghvljqhg#iru#djloh#whdpv1#Lw#lv#iuhh#iru#xs#wr#6#xvhuv#lq#Forxg#dqg#iru#43#xvhuv#lq#Vwdqgdorqh/#vr#li#|rx#zdqw#wr#jlyh#lw#d#jr#lq#|rxu#whdp#wkhq#zh#wrwdoo|#uhfrpphqg#lw1#|rx#kdyh#ilqlvkhg#wkh#iluvw#Txhvw/#qrz#lw“v#wlph#wr#uhghhp#|rxu#iluvw#sul}h1#Wkh#frgh#iru#wkh#iluvw#txhvw#lv#‟EhfdxvhFrgh†1#Jr#wr#wkh#Txhvw#Sdjh#dqg#xvh#wkh#frgh#wr#fodlp#|rxu#sul}h1#kwwsv=22zzz1mhweudlqv1frp2surpr2txhvw2

```



###  Decrypt

之前说的村好剑现在还真的派上用场了

- `Good luck! == Jrrg#oxfn$`
- `Qlfh$#Li#|rx#d...后面太长省略`

这两个的解密我是没有头绪的，于是无耻地偷看了提示` **Caesar cipher** `

凯撒加密还是很有名的，简单来说就是字符的移动位置变换。
伪代码如下

```scala
line.map(ch => (ch - (? - ?)).toChar)
```

实现完了以后大概是这个样子的



```scala
scala> val line = "Qlfh$#Li#|rx#duh#uhdglqj#wklv#|rx#pxvw#kdyh#zrunhg#rxw#krz#wr#ghfu|sw#lw1#Wklv#lv#rxu#lvvxh#wudfnhu#ghvljqhg#iru#djloh#whdpv1#Lw#lv#iuhh#iru#xs#wr#6#xvhuv#lq#Forxg#dqg#iru#43#xvhuv#lq#Vwdqgdorqh/#vr#li#|rx#zdqw#wr#jlyh#lw#d#jr#lq#|rxu#whdp#wkhq#zh#wrwdoo|#uhfrpphqg#lw1#|rx#kdyh#ilqlvkhg#wkh#iluvw#Txhvw/#qrz#lw“v#wlph#wr#uhghhp#|rxu#iluvw#sul}h1#Wkh#frgh#iru#wkh#iluvw#txhvw#lv#‟WkhGulyhWrGhyhors†1#Jr#wr#wkh#Txhvw#Sdjh#dqg#xvh#wkh#frgh#wr#fodlp#|rxu#sul}h1#kwwsv=22zzz1mhweudlqv1frp2surpr2txhvw2"
line: String = Qlfh$#Li#|rx#duh#uhdglqj#wklv#|rx#pxvw#kdyh#zrunhg#rxw#krz#wr#ghfu|sw#lw1#Wklv#lv#rxu#lvvxh#wudfnhu#ghvljqhg#iru#djloh#whdpv1#Lw#lv#iuhh#iru#xs#wr#6#xvhuv#lq#Forxg#dqg#iru#43#xvhuv#lq#Vwdqgdorqh/#vr#li#|rx#zdqw#wr#jlyh#lw#d#jr#lq#|rxu#whdp#wkhq#zh#wrwdoo|#uhfrpphqg#lw1#|rx#kdyh#ilqlvkhg#wkh#iluvw#Txhvw/#qrz#lw“v#wlph#wr#uhghhp#|rxu#iluvw#sul}h1#Wkh#frgh#iru#wkh#iluvw#txhvw#lv#‟WkhGulyhWrGhyhors†1#Jr#wr#wkh#Txhvw#Sdjh#dqg#xvh#wkh#frgh#wr#fodlp#|rxu#sul}h1#kwwsv=22zzz1mhweudlqv1frp2surpr2txhvw2

scala> line.map(ch => (ch - ('J'-'G')).toChar).mkString
res7: String = Nice! If you are reading this you must have worked out how to decrypt it. This is our issue tracker designed for agile teams. It is free for up to 3 users in Cloud and for 10 users in Standalone, so if you want to give it a go in your team then we totally recommend it. you have finished the first Quest, now it’s time to redeem your first prize. The code for the first quest is “TheDriveToDevelop”. Go to the Quest Page and use the code to claim your prize. https://www.jetbrains.com/promo/quest/


//上面是旧的，新的为
scala> line.map(ch => (ch - ('J'-'G')).toChar).mkString
res7: String = Nice! If you are reading this you must have worked out how to decrypt it. This is our issue tracker designed for agile teams. It is free for up to 3 users in Cloud and for 10 users in Standalone, so if you want to give it a go in your team then we totally recommend it. you have finished the first Quest, now it’s time to redeem your first prize. The code for the first quest is “TheDriveToDevelop”. Go to the Quest Page and use the code to claim your prize. https://www.jetbrains.com/promo/quest/

scala> val line = "Qlfh$#Li#|rx#duh#uhdglqj#wklv#|rx#pxvw#kdyh#zrunhg#rxw#krz#wr#ghfu|sw#lw1#Wklv#lv#rxu#lvvxh#wudfnhu#ghvljqhg#iru#djloh#whdpv1#Lw#lv#iuhh#iru#xs#wr#6#xvhuv#lq#Forxg#dqg#iru#43#xvhuv#lq#Vwdqgdorqh/#vr#li#|rx#zdqw#wr#jlyh#lw#d#jr#lq#|rxu#whdp#wkhq#zh#wrwdoo|#uhfrpphqg#lw1#|rx#kdyh#ilqlvkhg#wkh#iluvw#Txhvw/#qrz#lw“v#wlph#wr#uhghhp#|rxu#iluvw#sul}h1#Wkh#frgh#iru#wkh#iluvw#txhvw#lv#‟EhfdxvhFrgh†1#Jr#wr#wkh#Txhvw#Sdjh#dqg#xvh#wkh#frgh#wr#fodlp#|rxu#sul}h1#kwwsv=22zzz1mhweudlqv1frp2surpr2txhvw2"
line: String = Qlfh$#Li#|rx#duh#uhdglqj#wklv#|rx#pxvw#kdyh#zrunhg#rxw#krz#wr#ghfu|sw#lw1#Wklv#lv#rxu#lvvxh#wudfnhu#ghvljqhg#iru#djloh#whdpv1#Lw#lv#iuhh#iru#xs#wr#6#xvhuv#lq#Forxg#dqg#iru#43#xvhuv#lq#Vwdqgdorqh/#vr#li#|rx#zdqw#wr#jlyh#lw#d#jr#lq#|rxu#whdp#wkhq#zh#wrwdoo|#uhfrpphqg#lw1#|rx#kdyh#ilqlvkhg#wkh#iluvw#Txhvw/#qrz#lw“v#wlph#wr#uhghhp#|rxu#iluvw#sul}h1#Wkh#frgh#iru#wkh#iluvw#txhvw#lv#‟EhfdxvhFrgh†1#Jr#wr#wkh#Txhvw#Sdjh#dqg#xvh#wkh#frgh#wr#fodlp#|rxu#sul}h1#kwwsv=22zzz1mhweudlqv1frp2surpr2txhvw2

scala> line.map(ch => (ch - ('J'-'G')).toChar).mkString
res8: String = Nice! If you are reading this you must have worked out how to decrypt it. This is our issue tracker designed for agile teams. It is free for up to 3 users in Cloud and for 10 users in Standalone, so if you want to give it a go in your team then we totally recommend it. you have finished the first Quest, now it’s time to redeem your first prize. The code for the first quest is “BecauseCode”. Go to the Quest Page and use the code to claim your prize. https://www.jetbrains.com/promo/quest/
```

进入网址 https://www.jetbrains.com/promo/quest/ 输入验证码`BecauseCode`和邮箱即可领奖

### Prize 1

![](https://counter2015.com/picture/jb-quest-2020-7.jpg)

奖品是3个月JetBrains全家桶的订阅激活码，激活码在3个月内有效





而且还有接下来的预告

```text
Excited? You should be. The next quest will be at 1583924400 on our social media.

never odd or even,

The JetBrains Quest team
```

<del>这里我又犯蠢了，先是以为这串数字是电话号码，然后又以为没标时区</del>

```bash
$ date -d @1583924400 +"%F %H:%M:%S"
2020-03-11 19:00:00
```

那个时候估计我还没起床呢，非常期待之后的quest，一次愉快的解谜。

<p hidden> https://gist.github.com/counter2015/22703eba8039f6495b58197de4a370a9 <p>
-----

## Part 2

UPD: 2020-03-11 22:00

之前的时间问题，居然邮件是按当地时间发的，说好的Unix时间戳呢

新的一轮开始

![](https://counter2015.com/picture/jb-quest-2020-8.jpg)

### AD for DSL

这个很容易看出来， 有标点符号，最后三个字母是The，倒过来就行了。

```scala
scala> val line = ".spleh A+lrtC/dmC .thgis fo tuo si ti semitemos ,etihw si txet nehw sa drah kooL .tseretni wohs dluohs uoy ecalp a si ,dessecorp si xat hctuD erehw esac ehT .sedih tseuq fo txen eht erehw si ,deificeps era segaugnal cificeps-niamod tcudorp ehT"
line: String = .spleh A+lrtC/dmC .thgis fo tuo si ti semitemos ,etihw si txet nehw sa drah kooL .tseretni wohs dluohs uoy ecalp a si ,dessecorp si xat hctuD erehw esac ehT .sedih tseuq fo txen eht erehw si ,deificeps era segaugnal cificeps-niamod tcudorp ehT

scala> line.reverse
res0: String = The product domain-specific languages are specified, is where the next of quest hides. The case where Dutch tax is processed, is a place you should show interest. Look hard as when text is white, sometimes it is out of sight. Cmd/Ctrl+A helps.
```



> The product domain-specific languages are specified, is where the next of quest hides. The case where Dutch tax is processed, is a place you should show interest. Look hard as when text is white, sometimes it is out of sight. Cmd/Ctrl+A helps

大意是让我们去找一个DSL，它是Dutch税务指定产品，这时候又要求助搜索引擎了

![](https://counter2015.com/picture/jb-quest-2020-9.png)

其实这里就能基本确定这个产品的名字了，我们可以点进第一个链接具体查看。
![](https://counter2015.com/picture/jb-quest-2020-10.jpg)
没错，就是MPS，现在回到对应的产品页面，搜到MPS的网站地址。
https://www.jetbrains.com/mps/

找到含有Dutch tax的部分

![](https://counter2015.com/picture/jb-quest-2020-11.jpg)
这会打开一个新的PDF文件，还记得之前的提示吗
`Look hard as when text is white, sometimes it is out of sight. Cmd/Ctrl+A helps.`

![](https://counter2015.com/picture/jb-quest-2020-12.png)

很明显可以看出，右上角有隐藏的文字。

复制选中，发现空白处的文字为

```text
This is our 20th year as a company,
we have shared numbers in our JetBrains
Annual report, sharing the section with
18,650 numbers will progress your quest.
```

新的线索在年度报告里

### Report and Button

下一步继续搜年度报告

![](https://counter2015.com/picture/jb-quest-2020-13.jpg)

点进去后到达这一部分

![](https://counter2015.com/picture/jb-quest-2020-14.jpg)

发现这一小节正好是加起来为18650

![](https://counter2015.com/picture/jb-quest-2020-15.jpg)

这一小节里面能交互的地方不多，穷举法也能试出应该点击分享按钮

分享过程中有下一步的提示文本

```text
I have found the JetBrains Quest! Sometimes you just need to look closely at the Haskell language, Hello,World! in the hackathon lego brainstorms project https://blog.jetbrains.com/blog/2019/11/22/jetbrains-7th-annual-hackathon/ #JetBrainsQuest
```

进入[对应网站](https://blog.jetbrains.com/blog/2019/11/22/jetbrains-7th-annual-hackathon/)，找到相关图片的位置

![](https://counter2015.com/picture/jb-quest-2020-16.jpg)

这个文字从图上不好认，可以从源代码里找提示，找到的提示如下

```text
d1D j00 kN0w J378r41n2 12 4lW4Y2 H1R1N9? ch3CK 0u7 73h K4r33r2 P493 4nD 533 1f 7H3r3 12 4 J08 F0r J00 0R 4 KW357 cH4LL3n93 70 90 fUr7h3r @ l3457.
```

适箇就遈焱曐文（楧文限錠版钌），看懂牠需楆ー錠的厢象力，下靣是壹段飜譯

```text
did you know jetbrains is always hiring? check out the careers page and see if there is a job for you or for quest challenge to go further at least.
```



关于火星文的解法，其实在第一轮的获奖邮件里就有提示了

![](https://counter2015.com/picture/jb-quest-2020-21.jpg)

不过那句 `never odd or even` 没看懂什么意思

言归正传，之前的信息提示我们去招聘页面寻找下一步的线索

搜索`jetbrains hire` 到达  https://www.jetbrains.com/careers/jobs/ 

三个下拉菜单各自扫了一眼，唯一能放quest信息的只有`Any`了

任意一个修改成`Any`进入本轮信息页

![](https://counter2015.com/picture/jb-quest-2020-17.jpg)

下一步需要找到用于游戏开发的工具

### Game

照常，起手谷歌

![](https://counter2015.com/picture/jb-quest-2020-18.jpg)

好的进入这个产品首页

使用经典的[Konami作弊码](https://en.wikipedia.org/wiki/Konami_Code)

`↑↑↓↓←→←→BA`

神奇的事情发生了，出现了一个弹球游戏。

注意需要大写

![](https://counter2015.com/picture/jb-quest-2020-19.gif)

这个游戏对手残的我很不友好，还死了一次，幸好不用重头打过，游戏通过就可以看到领奖的地址和Key了。

这个代码换过一次，新的代码是`PlayGames`，领奖网页不变

### Prize 2

本轮的奖品是额外的3个月全家桶使用权

可以和之前的叠加，这样一共就有六个月了，JetBrains 牛逼！

新的邮件预告了下一次的活动

> You have passed the second quest and have proven yourself a worthy contender for our devious puzzles. We have one last quest in this series for you. It will be the ultimate quest, and it will secure your place in the halls of JetBrains Quest legend.
> A little @jetbrains bird will deliver the next clue – so do some birdwatching,



显然是说最后一轮会在推特上发布信息，期待最后一轮能给我们带来更多的乐趣吧，这轮玩得挺愉快的。



## Hidden Level

UPD: 2020-03-13

细心的人可能会发现，那就是两次领奖用的都是一个页面，假设领奖码本身就在前端，

是不是我们可以直接从前端页面破解出来呢？

强悍的群友在发现这个思路的一小时内就破解完成，我花了一天时间学习Chrome调试js代码，向群友学到了很多js的黑魔法后，也搞定了。



假设你已经知道了领奖网页，通过查看代码，发现判断Key正确与否的操作其实是在前端js进行的

我们可以想到，类似于`["key1", "key2"].continas(inputKey)`的方式来判断。

经过一段时间的排查，可以定位到如下代码部分。

![](https://counter2015.com/picture/jb-quest-2020-22.jpg)

这部分其实就是在数组里判断一个字符串是否存在，上面的数字是转换后的混淆

混淆很好解决，只要我们把光标定位在右中括号上，就知道这部分代码，复制下来丢到浏览器里运行下

![](https://counter2015.com/picture/jb-quest-2020-23.jpg)

看吧，结果出来了（真是怠惰啊 JetBrains）

如果第三轮还是同样的方式，就能一下子获取答案了。
<del>结果他真的第三轮答案就这样放出来了</del>

### Cheating

UPD: 2020-03-13 22:00

JetBrains还真是怠惰，同样的方式把密码破解出来了

但是估计是看前两问知道答案的人太多了，为了避免直接提交答案，修改了Key

前两问原先的答案` TheDriveToDevelop ` 和`GamesAreFun`

新的答案是

![](https://counter2015.com/picture/jb-quest-2020-24.jpg)

## Part 3

### Base64

UPD: 2020-03-14 14:00

![](https://counter2015.com/picture/jb-quest-2020-25.jpg)

新的一轮开始了

能用的常用解码方式就那么多，简单试了下，发现是Base64

```scala
scala> val line = "SGF2ZSB5b3Ugc2VlbiB0aGUgcG9zdCBvbiBvdXIgSW5zdGFncmFtIGFjY291bnQ/"
line: String = SGF2ZSB5b3Ugc2VlbiB0aGUgcG9zdCBvbiBvdXIgSW5zdGFncmFtIGFjY291bnQ/

scala> java.util.Base64.getDecoder().decode(line).map(_.toChar).mkString
res0: String = Have you seen the post on our Instagram account?
```

提示去[Instagrm account](https://www.instagram.com/jetbrains/)找线索

找到第一个图片

![](https://counter2015.com/picture/jb-quest-2020-26.jpg)

进去网页后没看到有用的东西，老规矩，查看网页源代码

![](https://counter2015.com/picture/jb-quest-2020-27.jpg)

提示我们下一步去[kotlin playground](https://jb.gg/kotlin_quest)查看线索

### 兀

playground上看起来又是一个需要解码的

```kotlin
fun main() {
   val s = "Zh#kdyh#ehhq#zrunlqj#552:#rq#wkh#ylghr#iru#wkh#iluvw#hslvrgh#ri#wkh#SksVwrup#HDS1#Li#zh#jdyh#|rx#d#foxh/#lw#zrxog#eh#hdv|#dv#sl1"

    val n: Int = TODO()
   for (c in s) {
       print(c - n)
   }
}
```

根据Quest 1的最后一步，显然这又是一个移位密码

还记得当初的密码吗，显然也是一个偏移量，因为要表达一个英文短句，那么显然空格是最多的，也就是说'#'对应的是空格

```scala
scala> val s = "Zh#kdyh#ehhq#zrunlqj#552:#rq#wkh#ylghr#iru#wkh#iluvw#hslvrgh#ri#wkh#SksVwrup#HDS1#Li#zh#jdyh#|rx#d#foxh/#lw#zrxog#eh#hdv|#dv#sl1"
s: String = Zh#kdyh#ehhq#zrunlqj#552:#rq#wkh#ylghr#iru#wkh#iluvw#hslvrgh#ri#wkh#SksVwrup#HDS1#Li#zh#jdyh#|rx#d#foxh/#lw#zrxog#eh#hdv|#dv#sl1

scala> s.map(ch => (ch - ('#' - ' ')).toChar).mkString
res4: String = We have been working 22/7 on the video for the first episode of the PhpStorm EAP. If we gave you a clue, it would be easy as pi.
```

怠惰的JetBrains看来是套路用完了，同样的招数居然用了两次

解码后可知，下一步的线索需要观看PhpStrom EAP第一章的视频，线索和圆周率有关

搜索后发现如下视频[地址](https://youtu.be/OtQuAr3n87c)

看评论，直接跳到对应时间点3:14(圆周率在这用上了)，同时把视频播放速度调节为0.25。

![](https://counter2015.com/picture/jb-quest-2020-28.jpg)

又是一个新的[链接](https://jb.gg/31415926)



### Questions

进入一个答题页面，下面是部分答案(这些都可以从第二问的年度报告里面找到)
参考 https://www.jetbrains.com/company/annualreport/2019/

-  What is the name of the newest JetBrains product?  答 Space
-  What year was JetBrains founded?  答 2000
-  Where is JetBrains headquarters?  答 捷克共和国
-  How many developers use our products? 答 8 million
-  What year was Kotlin introduced for the first time? 答 2011
-  Which country had the highest growing rate of downloads for 2019? 答 Burkina Faso
-  How many people founded JetBrains? 答 3
-  How many employees does JetBrains have? 答 1256
-  How much external funding have JetBrains received? 答 0
-  From the Forbes Top 100 digital companies, how many use our products? 答 95

终于答完了 
![](https://counter2015.com/picture/jb-quest-2020-29.jpg)

```text
Almost there! The last challenge is in the Tips of the Day of a specific
IntelliJ IDEA Community version from our latest build page in Confluence,
but… there is a catch. You have to know which version to look for. 
To find the build number, you need sight beyond sight:

. Not Everything Today Does All You Could Ask. 
Lessons Learned From Other Relevant Solutions,
Possibly Even Another Kind Emerge. Risking Sometimes 
Being Liberal Or Generous Proves Ordinary Simple 
Tests Infinitely More Annoying. Get Examining Hidden
Initial Designated Early Symbols. They Have Everything 
Needed, Except Xerox, To Completely Level Up Everything.
```

这段话里面，下半部分是线索，观察到每个字都是首字母大写，提取出来看看

```scala
scala> val line = ". Not Everything Today Does All You Could Ask. Lessons Learned From Other Relevant Solutions, Possibly Even Another Kind Emerge. Risking Sometimes Being Liberal Or Generous Proves Ordinary Simple Tests Infinitely More Annoying. Get Examining Hidden Initial Designated Early Symbols. They Have Everything Needed, Except Xerox, To Completely Level Up Everything."
line: String = . Not Everything Today Does All You Could Ask. Lessons Learned From Other Relevant Solutions, Possibly Even Another Kind Emerge. Risking Sometimes Being Liberal Or Generous Proves Ordinary Simple Tests Infinitely More Annoying. Get Examining Hidden Initial Designated Early Symbols. They Have Everything Needed, Except Xerox, To Completely Level Up Everything.

scala> line.split(" ").map(_.head).mkString
res5: String = .NETDAYCALLFORSPEAKERSBLOGPOSTIMAGEHIDESTHENEXTCLUE
```

句读(逗)一下`.net day call for speakers blog post image hides the next clue`

简单搜索后到达

 https://blog.jetbrains.com/dotnet/2020/02/13/jetbrains-net-day-online-2020-call-speakers/ 

这个网页，里面的图片url里藏着下一个线索

 https://d3nmt5vlzunoa1.cloudfront.net/dotnet/files/2020/02/you_are_looking_for_build_201-6303.png 

得知我们需要下载的版本号是`201-6003`

上部分得到的线索是`IntelliJ IDEA Community version from our latest build page in Confluence`

通过谷歌搜索`jetbrains confluence`我们来到了

 https://confluence.jetbrains.com/display/IDEADEV/IDEA+2020.1+Quest+Build+Edition 

600多MB还挺大的，下完了还得删掉

差点手滑删掉电脑上原来的IDEA

安装后打开，点的我手痛，终于在TIP of the Day 的某一条中发现了线索

![](https://counter2015.com/picture/jb-quest-2020-30.jpg)

要求计算斐波那契数列的第50 * 10 ^6 项，前四位和后四位连起来就是本次的key

这个有空专门写个文章[讲讲fib的计算](http://counter2015.com/2020/09/30/fibnacci/)套路吧 （Flag回收

抄了一个自己之前写的矩阵快速幂的板子

```scala
scala> :paste
// Entering paste mode (ctrl-D to finish)

object Fib {

  def fib(n: Int) = {
    type Matrix = Array[Array[BigInt]]


    def mul(a: Matrix, b: Matrix): Matrix = {
      Array(
      Array(a(0)(0) * b(0)(0) + a(0)(1) * b(1)(0), a(0)(0) * b(0)(1) + a(0)(1) * b(1)(1)),
      Array(a(1)(0) * b(0)(0) + a(1)(1) * b(1)(0), a(1)(0) * b(0)(1) + a(1)(1) * b(1)(1)))
    }

    @scala.annotation.tailrec
    def pow(a: Matrix, b:Matrix, n: Long): Matrix ={
      if (n == 0) b
      else {
        if (n % 2 == 1) {
          pow(mul(a,a), mul(b,a), n/2)
        } else pow(mul(a,a), b, n/2)
      }
    }

    val B: Matrix = Array(Array(1,0),Array(0,1))
    val a: Matrix = Array(Array(1,1),Array(1,0))
    val b: Matrix = Array(Array(0,1), Array(1,-1))
    if (n > 0) pow(a, B, n)(1)(0)
    else pow(b, B, 1-n)(0)(0)
  }
}


scala> val x = Fib.fib(50000000).toString
x: String = 46027134134331544532644523321018231246601789999278787750996232746285440008260845207292751373863063136109700395421340191690944799115396677587657065483882627093959104014474142751929723970951003729010625584111720279948233083391187074112128997051087841910469900302469028904301627684769178975443236869633635266383295864025985376849952974675314505841523224026295277530566720685827999764529550771403644356336116277430300068829944643897066291034660424393991878189124317540208284663285540477230578011892616747892072245412629172630839458603206256745366250260029509522623980648979192719884814190689613308024001032163978529782587710450786667556422574798722088143394375185386164394142278761603242028382026936677011173891244104499654098451229823156249181136506503816303657652206166836283354070250584...

scala> x.take(4) + x.takeRight(4)
res1: String = 46023125
```

上面的代码整理了下，用我的破机器用非常trival的运行参数来运行测试了下时间
```scala
$ time scala fib.scala
46023125
real	2m6.698s
user	2m21.891s
sys	0m3.344s

$ scalac fib.scala
$ scala Fib
46023125

real	1m52.475s
user	2m0.578s
sys	0m2.344s
```



官方题解发布了，解答非常tricky

```python
import math

def last_fib_digits(fib_number, last_digits):
   prev, cur = 0, 1
   q = 10 ** last_digits
   while fib_number > 0:
       prev, cur = cur, prev + cur
       fib_number -= 1
       cur %= q
   return prev

def first_fib_digits(fib_number):
   phi = (math.pow(5.0, 0.5) + 1) / 2
   logF = fib_number * math.log10(phi) - 0.5 * math.log10(5.0)
   return math.pow(10.0, logF - int(logF))

print(last_fib_digits(50000000, 4))

print(first_fib_digits(50000000))
```

通用用我的破烂机器跑了一遍

```python
time python fib.py
3125
4.602713419459862

real	0m6.020s
user	0m5.609s
sys	0m0.047s
```

时间差距这么大的原因，是因为这个做法并没有*真正*计算出fib(50000000)的结果，而是耍了一个花枪

这个花枪从数学公式可以这样看

令`an`为斐波那契数列的第n项，可知通项公式为
$$
a_n = \frac{(\frac{1+\sqrt5}{2})^n-(\frac{1-\sqrt5}{2})^n}{\sqrt5}
$$
我们想要求解`a50000000`的前四位数字，其实不需要获取准确的结果，估算也行

考虑这么一个式子
$$
b_n = {(\frac{1-\sqrt5}{2})^n}
$$

我们可以把这部分直接舍去，因为当n变大时，bn会趋近于0

令
$$
c_n = log_{10}{b_n} = nlog_{10}{\frac{1+\sqrt5}{2}} - \frac{log_{10}{5}}{2}
$$
我们求前4位数字，可以舍去`Cn`里的整数部分，也就是求
$$
10 ^ {c_n - \lfloor c_n \rfloor}
$$
同样的思路实现一遍

```scala
object Fib {
  def main(args: Array[String]): Unit = {
    val n = 50000000
    
    def last: Int = {
      var (a, b, number) = (0, 1, n)
      val mod = 10000
      while (number > 0) {
        val temp = a + b
        a = b
        b = temp % mod
        number -= 1
      }
      a
    }

    def first = {
      val x = (math.sqrt(5) + 1) / 2
      val lx = n * math.log10(x) - 0.5 * math.log10(5)
      math.pow(10, lx - math.floor(lx))
    }

    print(first.toString.filter(_.isDigit).take(4) + last)
  }
}
```

再次测试

```shell
$ scalac fib.scala
$ time scala Fib
46023125
real	0m1.271s
user	0m1.078s
sys	0m0.344s
```





### Prize 3

至此，所有的quest就解开了，不过本轮的奖励有点一言难尽

![](https://counter2015.com/picture/jb-quest-2020-31.jpg)





## All Prizes

以下是完整答案。

网址： https://www.jetbrains.com/promo/quest/ 

Code:

- BecauseCode（TheDriveToDevelop 已废弃）
- PlayGames (GamesAreFun已废弃)
- 46023125



## 致谢

1. [JetBrains Quest](https://mihajlonesic.gitlab.io/archive/jetbrains-quest/)
2. [如何白嫖3个月的JetBrains全家桶（包括Java神器IDEA)](https://miion.me/2020/03/10/%E5%A6%82%E4%BD%95%E7%99%BD%E5%AB%963%E4%B8%AA%E6%9C%88%E7%9A%84JetBrains%E5%85%A8%E5%AE%B6%E6%A1%B6/)
3. [JetBrains](https://www.jetbrains.com/)
4. [JetBrains 解谜第二波有人在做吗？](https://www.v2ex.com/t/651986)
5. 群友们