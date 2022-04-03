
# github pages微博图床图片无法显示

title: github pages微博图床图片无法显示
date: 2019-04-29 21:35:24
tags: 图床
categories: [Bugfix]

---

今天发现博客上的图片链接都挂了，大部分显示不出来，小部分能用

![](https://counter2015.com/picture/bug-2-1.png)



首先开始排查是否是来源微博图床挂了的可能性，发现在网页中能正常访问。

接着排查浏览器的可能性，发现所有的浏览器都不能打开。

然后开始检索问题关键词`github pages 无法加载图片 this request content cant`

进入github pages[帮助文档](<https://help.github.com/en/articles/securing-your-github-pages-site-with-https>)

上面指出可能是由于https网页中调用http资源导致的问题。但是之前图片能正常显示，所以这个变动一定是由外部最近的原因导致的。

全部替换链接为https有点麻烦，偶然发现了问题所在**“微博限制图片外链”**

按照这个关键字，一下就定位到了[原因](<https://github.com/suxiaogang/WeiboPicBed/issues/122>)<del>免费的才是最贵的</del>了。

那么接下来就是解决办法了。首先想到的是将图片放置在本地，同步博客内容时手动上传

首先想到的是这个方法，但是这个办法太蠢了，每次都要手动往文件夹里添加图片。

但是用其他图床也不靠谱，随时可能禁止用户使用免费外链，这种问题应该hexo开发者有人考虑过，于是开始检索方案。

查到这个[文档](<https://hexo.io/zh-cn/docs/asset-folders.html>)

> 对于那些想要更有规律地提供图片和其他资源以及想要将他们的资源分布在各个文章上的人来说，Hexo也提供了更组织化的方式来管理资源。这个稍微有些复杂但是管理资源非常方便的功能可以通过将 `config.yml`文件中的 `post_asset_folder` 选项设为 `true` 来打开。

但是这个命令是用`hexo new [layout] <title>`来触发创建同名资源文件夹，这有两个问题

- 我使用习惯用本地markdown编辑器来创建新的博客，而不是hexo命令
- 创建的规则过于僵硬 

还好之前上传的记录都能找到，把图片一个个下载下来再来搞吧。

> 资源（Asset）代表 `source` 文件夹中除了文章以外的所有文件，例如图片、CSS、JS 文件等。比方说，如果你的Hexo项目中只有少量图片，那最简单的方法就是将它们放在 `source/images` 文件夹中。然后通过类似于 `![](/images/image.jpg)` 的方法访问它们。



不负责任的猜测下：由于下载刷新了图片的请求时间，微博图床按照这个来判断是否对外提供服务。

总之，现在已经把图片都转移到本地了,有空的时候慢慢地将替换掉微博链接。



后来经他人指点发现一个更简单的方法：

- 在github上新建一个仓库
- 在仓库设置中做如下修改

![](https://counter2015.com/picture/bug-2-2.png)

这样一来就能方便的用github来做图床了

```
# 在markdown中添加类似链接以插入图片
![](https://counter2015.com/picture/bug2-2.png)
```



UPD: 后来发现这个办法其实在github的[官方课程](<https://lab.github.com/githubtraining/github-pages>)里有。