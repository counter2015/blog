
# 单引号显示为中文引号

title: 单引号显示为中文引号
date: 2019-01-19 20:10:24
tags: hexo
categories: [Bugfix]

----

在写markdown的时候，发现了代码里的引号咋变成中文引号了。

这样直接复制代码是会报错的，于是开始定位问题原因。

出现问题的代码如下：

```html

<details>
  <summary>端口为7000节点的完整配置文件示例如下(去除注释): </summary>
  <p></p>
  <pre><code>
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
  </code>  </pre>
</details>
```

这段代码是为了在markdown里面引用一个配置文件而使用的，由于配置文件太长，影响文章的整体展示效果，于是找了个方法做成点击后显示。结果就出问题了，英文引号都变成中文引号。



第一时间就想到，是不是引入html代码的问题，于是写了个测试文件markdown，来测试引号的显示问题。

```markdown


单纯的引号 ”“
rt ""

\`\`\`
代码里的引号 ”“
rt ""
\`\`\`


<details>
  <summary>测试文件: </summary>
  <p></p>
  <pre><code>
测试文本
英文引号""
中文引号“”
en ""
zh ”“
lenrzh "”
lzhren “"
  </code></pre>
</details>
```

不测不知道，一测真奇妙，所有的引号都会被显示成英文引号。这就有点头疼了，看来之前的文章也有这个问题，只是由于引号出现的频率不高所以没有被发现。



排除了html代码引起的问题，下一个排查的是hexo 主题和插件，但是我对插件也是知其然不知其所以然，就先从hexo主题查起吧，于是先切换了个主题。



切换成默认主题`landscape`还是这个问题，看来问题发生在更底层的位置。<del>因为频繁地切换主题导致hexo渲染出错，主页一团糟，后来`hexo clean`解决</del>



灵光一闪，是不是只有电脑端才有这个问题，于是用手机登录了下，嗯，看起来正常多了，但是，当复制对应文本发送到电脑查看时，发现还是中文引号，只是眼睛欺骗了我，(￣_￣|||)



好了，全平台报错，现在我有一半以上的可能性确定问题不在第三方组件，而在hexo上了，于是打开github，找到hexo的issue区，发现了不是我一个人掉坑里，欣慰

[满屏幕的中文十分亲切](https://github.com/hexojs/hexo/issues/1981)

[另外一个更详细的链接](<https://github.com/theme-next/hexo-theme-next/issues/462>)





> @**swj1442291549** 我看了一下，渲染的 rendered font 是 “Microsoft YaHei”。而 CSS 中的字体似乎都是 macOS 上独有的，导致浏览器使用才做系统的默认字体。而 “Microsoft YaHei” 对于引号的处理就是全角的。所以我改了一下 CSS 强制换了一个 font-family，就解决了
> @**tommy351** 如果用的是 marked renderer 的話，可以試試看關掉 smartypants？
>
> ```yml
> marked:
> smartypants: false
> ```



通过修改hexo的`_config.yml`成功解决。

关于这个markdown的渲染引擎。

> - smartLists - Use smarter list behavior than the original markdown.



{% spoiler smartLists自作聪明，当前版本hexo尚未修复这个问题，看来注意到的人不多 %}

