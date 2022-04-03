# giter8简单介绍

title: giter8简单介绍
date: 2019-02-01 15:55:24
tags: giter8
categories: [技术]

---

## 前言

Giter8是一个用来生成scala项目模板的工具。

在之前[scalatra的介绍](http://counter2015.com/2019/01/04/scalatra1/)中我们提到过[giter8](http://www.foundweekends.org/giter8/index.html),这里就再往前迈一步，讲解giter8的基础使用方法。

官网：http://www.foundweekends.org/giter8/index.html

由于众所周知的网络问题，部分操作可能需要科学上网。

### 准备工作



**你可以先尝试<a href="#1">直接安装</a>，如果不行再尝试下面开启代理的方法**

部分包的下载需要开启代理，而这需要对下载命令配置代理，这里使用的是[proxychains-ng](https://github.com/rofl0r/proxychains-ng)

{% spoiler 这个名字的品味令人无语，ng next/new generation，我一开始还以为是no good。作者是不打算出3了？要不然叫 ***nng***  (ˉ▽ˉ；)... %}

```shell
# proxychains的下载
$ git clone https://github.com/rofl0r/proxychains-ng.git
$ cd proxychains-ng

# 设置安装位置和配置文件存放位置
$ ./configure --prefix=/root/bin/proxychains-ng --sysconfdir=/etc
$ make && make install-config
$ cp proxychains4 /root/bin/proxychains-ng/


# 修改配置文件
$ vim /etc/proxychains.conf
```



将最后一行的配置修改为 `socks5 <ipv4-address> <proxy-port>`

- sock5, 使用的协议名称
- `ipv4-address`, 开启代理服务器的ip地址
- `proxy-port`, 设置的代理端口

这里我使用的是本地代理，没有设置用户名和密码，端口选择的是`1080`,也就是说，我写的是

`socks5 127.0.0.1 1080`

下面是我为了方便个人使用配置的路径，可以按自己喜好随意修改。

```shell
$ vim ~/.bashrc
  export PATH=/root/bin/proxychains-ng/bin:$PATH
$ source ~/.bashrc
```



```shell
# Proxychains简单示例
$ proxychains4 ping google.com
PING google.com(hkg07s30-in-x0e.1e100.net (2404:6800:4005:811::200e)) 56 data bytes

```

## <a name="1">安装</a>



```shell
# 首先安装giter8的依赖conscript
$ curl https://raw.githubusercontent.com/foundweekends/conscript/master/setup.sh >> setup.sh
$ proxychains4 sh setup.sh 

# 使用conscript安装giter8
# 直接安装时不需要`proxychains4`
$ cs foundweekends/giter8

# 验证g8安装
$ g8
# g8的使用方法如下
Error: Missing argument <template>
g8 0.12.0-M1
Usage: g8 [options] <template>

  <template>               git or file URL, or github user/repo
  -b, --branch <value>     Resolve a template within a given branch
  -t, --tag <value>        Resolve a template within a given tag
  -d, --directory <value>  Resolve a template within a given directory
  -o, --out <value>        Output directory
  -f, --force              Force overwrite of any existing files in output directory
  --version                Display version number
  --paramname=paramval  Set given parameter value and bypass interaction

EXAMPLES

Apply a template from github
    g8 foundweekends/giter8

Apply using the git URL for the same template
    g8 git://github.com/foundweekends/giter8.git

Apply template from a remote branch
    g8 foundweekends/giter8 -b some-branch

Apply template from a remote tag
    g8 foundweekends/giter8 -t some-tag

Apply template from a directory that exists in the repo
    g8 foundweekends/giter8 -d some-directory/template

Apply template into an output directory
    g8 foundweekends/giter8 -o output-directory

Apply template from a local repo
    g8 file://path/to/the/repo

Apply given name parameter and use defaults for all others.
    g8 foundweekends/giter8 --name=template-test
```



## 构建自定义模板





git8官方推荐使用[CC0](https://creativecommons.org/publicdomain/zero/1.0/)的协议，这样别人可以引用你的模板而不用担心侵权问题（当然你也可以选择其他License）

模板的布局有两种

- src layout: 存在`src/main/g8`
- root layout: 不属于上一种情况时



这里以src layout为例

*要想创建模(di)板(gui)，先要创建模(di)板(gui)*

启动新模板项目的简单方法是使用专门为此目的制作的Giter8模板：

```shell
$ g8 foundweekends/giter8.g8

name [My Template Project]: g8test.g8
giter8_version [0.12.0-M1]: 
sbt_version [1.2.8]: 

Template applied in /root/tmp/./gtest

$ cd g8test.g8
# 建立好的模板项目，目录结构如下
$ tree
.
├── README.markdown
├── build.sbt
├── project
│   ├── build.properties
│   └── plugins.sbt
└── src
    └── main
        └── g8
            ├── build.sbt
            ├── default.properties
            ├── project
            │   └── build.properties
            └── src
                └── main
                    └── scala
                        └── Stub.scala

```



`default.properties`使用Java属性文件的格式来定义模板字段其默认值。

```shell
$ cat src/main/g8/default.properties 
name=My Something Project
description=Say something about this template.

```

当你用这个模板新建项目时就会用到这里定义的字段。

g8模板的默认存取位置为github。

我们可以先试着把这个模板丢到github上去，再下下来看看效果<del>听起来咋这么无聊呢</del>

省略在github上创建仓库的步骤，仓库命名建议以`.g8`为后缀

```shell
$ git init
$ git add .
$ git commit -m "first commit"
$ git remote add origin https://github.com/counter2015/g8test.g8.git
$ git push -u origin master
```

换个目录再把它下下来

```shell
$ cd ..
$ g8 counter2015/g8test.g8
name [My Something Project]: step1
$ cd step1

$ tree
.
|-- build.sbt
|-- project
|   `-- build.properties
`-- src
    `-- main
        `-- scala
            `-- Stub.scala
```

可以看出，我们从模板中生成的程序，正是`g8test.g8`中目录`src/main/g8`下的文件(除了`default.properties`)

还能看出，`My Something Project`的交互提示，正是`src/main/g8/default.properties `的内容

下面来尝试使用`default.properties`，里面使用的技术有:

- [StringTemplate](https://www.stringtemplate.org/)
- [Scalasti](http://bmc.github.com/scalasti/)

我们暂且不需要关心其中的细节

我们先返回最开始的目录，修改`g8test.g8`项目中的`default.properties`

```shell
$ cd g8test.g8
# 修改部分属性
$ vim src/main/g8/default.properties 
name=step2
description=a simple project
words=Hello giter8!


$ git add -A
$ git commit -m "modify some properties"

$ vim src/main/g8/src/main/scala/Stub.scala
object Main {
  def main(args: Array[String]): Unit = {
    println($words$)
  }
}


$ mv src/main/g8/src/main/scala/Stub.scala src/main/g8/src/main/scala/Main.scala

$ git add -A
$ git commit -m "modify a scala file"

$ git push

```



准备完成，我们重新试下有什么变化



```shell
$ g8 counter2015/g8test.g8
name [step2]: 
words [Hello giter8!]: "hi, there!"

$ cd step2
$ sbt run
hi, there!

$ cat src/main/scala/Main.scala 
object Main {
  def main(args: Array[String]): Unit = {
    println("hi, there!")
  }
} 
```



> Notes: 
>
> - `description`字段暂时还[无法使用](https://github.com/foundweekends/giter8/issues/99)
> - 暂不支持对文件名使用模板，如`src/main/scala/$classname$.scala`，这种会直接报错









## 总结

花了5小时将原来20分钟的事情简化到了2分钟

<del>5*60/18≈12</del>



