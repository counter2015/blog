# Github Actions 实战—— 下载每日Bing图片

title: Github Actions 实战——下载每日Bing图片
date: 2020-05-10 19:00:30
tags: [Github Actions, Bing] 
categories: [技术]

------



Github Actions 出来也有一段时间了，内测时候就开始看了，现在才终于有空用上。

这里将会用一个下载每日Bing的图片的例子，来说明Github Actions的使用

![](https://cn.bing.com/th?id=OHR.BarnOwlMigration_ZH-CN8579914070_1920x1080.jpg)

既然要下载图片，首先我们需要一个下载的脚本

通过F12分析Bing的网页，可以很简单地发现对应的图片链接

```html
<!DOCTYPE html><html lang="zh"><script type="text/javascript" >//<![CDATA[
si_ST=new Date
//]]></script><head><link id="bgLink" rel="preload" href="/th?id=OHR.BarnOwlMigration_ZH-CN8579914070_1920x1080.jpg&amp;rf=LaDigue_1920x1080.jpg&amp;pid=hp" as="image" /><link rel="preload" href="/sa/simg/hpc27.png" as="image" /><meta content="text/html; charset=utf-8" http-equiv="content-type"/><script type="text/javascript">//<![CDATA[
```

那。。。就用Python爬虫爬下来吧？

其实不需要引入爬虫库，这个链接只要简单用正则处理下就能拿到了



在上面这段文本中，我们需要的其实是第一个`href`标签栏后面的链接地址

```shell
/th?id=OHR.BarnOwlMigration_ZH-CN8579914070_1920x1080.jpg&amp;rf=LaDigue_1920x1080.jpg&amp;pid=hp
```

`&`符号后面的url可以省略

```shell
/th?id=OHR.BarnOwlMigration_ZH-CN8579914070_1920x1080.jpg
```

那么完整的图片url为

```shell
http://bing.com/th?id=OHR.BarnOwlMigration_ZH-CN8579914070_1920x1080.jpg
```

其中`BarnOwlMigration`是图片的描述信息`畜禽迁徙`,可以用作下载的图片名称

那么我们需要如下函数

```scala
// 从bing.com 获取到link链接，
// e.g. /th?id=OHR.BarnOwlMigration_ZH-CN8579914070_1920x1080.jpg
def getLink: String = ???

// 从link中获取图片名称
// e.g. BarnOwlMigration
def imageName(url: String): String = ???

// 将指定url中的图片下载到filename所在的路径
def fileDonloader(url: String, filename: String): Unit = ???
```

前两个函数是对字符串处理，后面的步骤可以使用Scala自带的`scala.sys.process`库来调用系统命令

完整代码如下，不到50行

```scala
import java.io.File
import java.net.URL


import scala.io.Source
import scala.language.postfixOps
import scala.language.implicitConversions
import scala.sys.process._


def fileDonloader(url: String, filename: String): Unit = {
  new URL(url) #> new File(filename) !!
}

val header = "https://bing.com/"

def getLink: String = {
  val src = Source.fromURL("https://bing.com")
  val lines = src.mkString
  val pattern = """(th\?id=OHR.*?jpg)""".r
  val link = pattern.findFirstIn(lines).get
  println(s"link=$link")
  link
}

val tail = getLink

val url = header + tail

val imageNameExtractPattern = ".*id=OHR\\.(.*?)_.*".r

def imageName(url: String): String = url match {
  case imageNameExtractPattern(name) => name
  case _ => "noname"
}

val filetype = ".jpg"
val folderName = "images/"
val filename = folderName + imageName(url) + filetype

fileDonloader(url, filename)
println(s"download $url to $filename")
```

这里我稍微偷了点懒，没有考虑网络和解析的异常情况(Github和微软不是内网吗，那不是稳的一比<del>才怪现在还在AWS上</del>）。

接下来就是Github Action的编写了，这一部分可以参考[官方文档](https://help.github.com/en/actions)

其实就是用DSL来写控制命令

因为是日更新，所以稍微调整下对应的触发方式`cron`

 https://help.github.com/en/actions/reference/events-that-trigger-workflows#scheduled-events-schedule 

```yaml
on:
  schedule:
    # runs at 00:00 UTC every day
    - cron:  '0 0 * * *'
```

我们来简单梳理下所需要的步骤

- 设置运行环境(OS, jdk, scala)
- 将代码clone下来
- 运行图片下载脚本
- 维护README中的展示页面，修改时间和图片信息
- Commit & Push



对应的workflow配置如下

```yaml
name: Collect Bing.com daily images

on:
  schedule:
    - cron:  '0 */10 * * *'

jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
    # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
    - uses: actions/checkout@v2
      
    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8

    - name: Set up Scala 2.13.2
      run: |
        wget https://downloads.lightbend.com/scala/2.13.2/scala-2.13.2.deb
        sudo dpkg -i scala-2.13.2.deb && sudo apt install -f
        scala --version
        
    - name: Download image
      run: scala download.scala

    - name: Upadte README
      run: |
        image=`ls -l images | grep jpg | head -1 | awk '{print $9}'`
        echo latest image is $image
        sed -i "s/last update:.*/last update:`date +"%F %H:%M:%S"` UTC/g" README.md
        sed -i "s#\!\[\](images/.*)#\!\[\](images/$image)#g" README.md

    - name: Commit
      run: |
        git config user.name counter2015
        git config user.email voidcounter@gmail.com
        git status
        git add images/*.jpg README.md
        git commit -m "auto commit from github actions" 
        
    - name: Push
      uses: ad-m/github-push-action@master
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}

```

完整的仓库地址在[此处](https://github.com/counter2015/bing-daily-images)

你可以在这里直接查看[在线demo](https://counter2015.com/bing-daily-images/)

P.S. 

- 由于部署到Github Actions上，所以不同地区看到的网页会有所不同
- `secrets.GITHUB_TOKEN`要想生效，需要 https://github.com/settings/tokens 处添加名为`GITHUB_TOKEN`的token，并且分配`admin:repo_hook,repo,workflow`的权限



## 更新

后来发现有直接的API可以调用，不用苦哈哈的去正则

```bash
$ curl "https://www.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1" | python -m json.tool
{
    "images": [
        {
            "startdate": "20200514",
            "fullstartdate": "202005141600",
            "enddate": "20200515",
            "url": "/th?id=OHR.NorthRimOpens_ZH-CN9513300299_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp",
            "urlbase": "/th?id=OHR.NorthRimOpens_ZH-CN9513300299",
            "copyright": "\u4eceToroweap Overlook\u4fef\u77b0\u5927\u5ce1\u8c37\u548c\u79d1\u7f57\u62c9\u591a\u6cb3\uff0c\u4e9a\u5229\u6851\u90a3\u5dde\u5927\u5ce1\u8c37\u56fd\u5bb6\u516c\u56ed (\u00a9 Matteo Colombo Travel Photo/Shutterstock)",
            "copyrightlink": "https://www.bing.com/search?q=%E5%A4%A7%E5%B3%A1%E8%B0%B7%E5%9B%BD%E5%AE%B6%E5%85%AC%E5%9B%AD&form=hpcapt&mkt=zh-cn",
            "title": "",
            "quiz": "/search?q=Bing+homepage+quiz&filters=WQOskey:%22HPQuiz_20200514_NorthRimOpens%22&FORM=HPQUIZ",
            "wp": false,
            "hsh": "d9e6fdc360677175dbd084248898cf45",
            "drk": 1,
            "top": 1,
            "bot": 1,
            "hs": []
        }
    ],
    "tooltips": {
        "loading": "\u6b63\u5728\u52a0\u8f7d...",
        "previous": "\u4e0a\u4e00\u4e2a\u56fe\u50cf",
        "next": "\u4e0b\u4e00\u4e2a\u56fe\u50cf",
        "walle": "\u6b64\u56fe\u7247\u4e0d\u80fd\u4e0b\u8f7d\u7528\u4f5c\u58c1\u7eb8\u3002",
        "walls": "\u4e0b\u8f7d\u4eca\u65e5\u7f8e\u56fe\u3002\u4ec5\u9650\u7528\u4f5c\u684c\u9762\u58c1\u7eb8\u3002"
    }
}

```

其实只要用bash直接能下载

```bash
tail=`curl "https://www.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1" | grep -oP "th\?id=.*?jpg"`
image_name=`echo $tail | grep -oP 'OHR\.\K([a-zA-Z]+)'`
wget -O "images/${image_name}.jpg" https://bing.com/${tail}
```

---
如果说为了刷 commit, 可以把提交是的用户名和邮箱设置成自己
否则的话，我们可以修改下提交到 [`Github Actions bot`](https://github.community/t/github-actions-bot-email-address/17204) 名下
```yaml
    - name: Commit
      run: |
        git config user.name "github-actions[bot]"
        git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
        git status
        git add images/*.jpg README.md
        git commit -m "auto commit from github actions" 
```

## 参考链接

1. [必应][1]
2. [github push action][2]
3. [github actions doc][3]
4. [Github Actions 教程： 运行Python代码并部署到远程仓库][4]
5. [Is there a way to get bings photo of the day][5]



[1]: https://bing.com
[2]: https://github.com/ad-m/github-push-action
[3]:  https://help.github.com/en/actions
[4]: https://www.cnblogs.com/marsggbo/p/12090703.html
[5]:  https://stackoverflow.com/questions/10639914/is-there-a-way-to-get-bings-photo-of-the-day