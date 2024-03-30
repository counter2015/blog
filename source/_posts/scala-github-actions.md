# 使用 Github Actions 构建 Scala 项目

title: 使用 Github Actions 构建 Scala 项目
date: 2024-03-30 17:27:44
tags: [Github Actions, Scala]
categories: [技术]

---

Github Actions 这个功能距离首次发布到现在也有5年左右了，现在已经是各大开源项目的标配了。

我们在 Scala 项目中也能通过使用 Github Actions 来帮助自动化测试以及更新依赖。

有了各种 Actions 的帮助，可以为我们的项目添加各种<del>酷炫的</del>徽章

[![](https://img.shields.io/badge/Scala_Steward-helping-blue.svg?style=flat&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAQCAMAAAARSr4IAAAAVFBMVEUAAACHjojlOy5NWlrKzcYRKjGFjIbp293YycuLa3pYY2LSqql4f3pCUFTgSjNodYRmcXUsPD/NTTbjRS+2jomhgnzNc223cGvZS0HaSD0XLjbaSjElhIr+AAAAAXRSTlMAQObYZgAAAHlJREFUCNdNyosOwyAIhWHAQS1Vt7a77/3fcxxdmv0xwmckutAR1nkm4ggbyEcg/wWmlGLDAA3oL50xi6fk5ffZ3E2E3QfZDCcCN2YtbEWZt+Drc6u6rlqv7Uk0LdKqqr5rk2UCRXOk0vmQKGfc94nOJyQjouF9H/wCc9gECEYfONoAAAAASUVORK5CYII=)](https://scala-steward.org)
![](https://github.com/counter2015/tccc/actions/workflows/scala-steward.yaml/badge.svg)
[![](https://codecov.io/gh/counter2015/tccc/branch/master/graph/badge.svg)](https://codecov.io/gh/counter2015/tccc)
[![](https://img.shields.io/badge/scala-%3E%3D%203.4.0-8892BF.svg)](https://www.scala-lang.org/)
[![](https://img.shields.io/badge/jdk-%3E%3D%2021-8892BF.svg)](https://openjdk.org/)


## 和 codecov/coverage 集成

[sbt-scoverage](sbt-scoverage) 是一个用来提供 Scala 代码测试覆盖率的 sbt 插件。

在项目中引入很简单：

```
// in project/plugins.sbt
addSbtPlugin("org.scoverage" % "sbt-scoverage" % "2.0.11")
```

插件安装完成后，就可以通过运行`sbt coverageReport` 在本地生成测试覆盖率的报告了，报告会生成在`target/scala-x.y.z/scoverage-report/index.html`的位置，可以从浏览器中打开查看。



sbt-scoverage 可以和多个测试平台集成，这里我们选用的是最常见的 `Codecov`

通过创建对应的 Github Actions workflow, 我们可以在推送代码后自动触发测试流程，并且生成测试覆盖率报告，并把它上传到 Codecov 网站上。

这个实现起来很方便，详细步骤可以参考这个[示例项目](https://github.com/codecov/example-scala)

首先我们需要编写 workflow 的文件，在项目的`.github/workflows/test.yml`中添加如下配置

```yaml
name: Test workflow

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    name: Test scala
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'adopt'
          java-version: '21'
      - name: Run tests
        run: sbt coverage test coverageReport
      - uses: codecov/codecov-action@v4
        with:
          fail_ci_if_error: true
          token: ${{ secrets.CODECOV_TOKEN }}
```

这个 workflow 设定成在 push, pull request 的时候触发，使用 ubuntu 的机器来进行测试。

整体流程分为这么几个步骤，首先把仓库的代码拉取到本地，然后安装 adopt-openjdk 21, 接着使用 sbt 运行测试并生成测试率报告，最后把生成的报告上传到 codecov 上面。

这里唯一需要注意的配置项是 `CODECOV_TOKEN`，这个 token 需要先在 codecov 上配置，具体步骤可以参考[官方文档](https://docs.codecov.com/docs/quick-start#step-2-get-the-repository-upload-token)

复制好对应项目的 `CODECOV_TOKEN` 后，需要我们在自己的对应仓库里做对应的配置

具体的位置在：`Settings(项目右上角) -> Secrets and variables(左侧目录) -> New repository secret(绿色按钮)`

然后把新建一个 `CODECOV_TOKEN` 的 secrets, 并把之前复制好的 token 粘贴过来。

![](https://counter2015.com/picture/github-actions-2-1.png)

如此一来，每次提交代码后，Github Actions 都会帮我们跑一遍自动化测试，我们可以在项目的 `Actions` 里查看对应的结果。

之后我们可以在项目的 README.md 中添加徽章

`[![codecov](https://codecov.io/gh/YOUR_ACCOUNT/YOUR_REPO/branch/YOUR_BRANCH/graph/badge.svg)](https://codecov.io/gh/YOUR_ACCOUNT/YOUR_REPO)`

比如我的用户名是 `counter2015`, 对应的仓库是 `tccc`, 分支是 `master`, 那么具体的链接就是

`[![codecov](https://codecov.io/gh/counter2015/tccc/branch/master/graph/badge.svg)](https://codecov.io/gh/counter2015/tccc)`

渲染出来的结果是这样的
[![](https://codecov.io/gh/counter2015/tccc/branch/master/graph/badge.svg)](https://codecov.io/gh/counter2015/tccc)


## 和 scala-steward 集成

scala-steward 是 fthomas 开发的一个项目，用来自动更新 Scala 项目中过时的版本依赖，Scala 的官方博客早在2019年就有[介绍](https://www.scala-lang.org/blog/2019/07/10/announcing-scala-steward.html)

当前是由 `VirtusLab` 维护在这个仓库地址`https://github.com/VirtusLab/scala-steward-repos`

曾经有一段时间，由于 scala-steward 的账户自动提交的 pull request 过多，导致其 [Github 主页](https://github.com/scala-steward)加载超时，现在应该是修好了。



在 Github 上使用 scala-steward 主要有两种方式：

1. 通过给 VirtualLab 维护的仓库列表提交 PR, 把自己的仓库添加到需要自动更新依赖的列表中，就可以使用社区的 bot 帮你维护啦，这个[文件](https://github.com/VirtusLab/scala-steward-repos/blob/main/repos-github.md) 目前有1622个仓库
2. 自行创建并维护一个 scala-steward-action 的应用



这里主要介绍第二种方式，这种方式可以在自己的账户或者组织下添加一个新的 scala-steward 应用。

创建的具体步骤可以参考 [scala-steward-action](https://github.com/scala-steward-org/scala-steward-action) 的文档，步骤大致可以分为一下几步：

1. 创建一个新的 Github App
2. 安装 App
3. 复制 **App ID, the App private key and the installation ID**
4. 创建仓库 Secrets
5. 创建一个新的 Scala steward Github Actions Workflow
6. 使用 Scala Steward 来搞定依赖更新

这里详细讲一下容易找不到操作位置的步骤3。

我们需要以下几个配置

- APP_ID
- APP_INSTALLATION_ID
- APP_PRIVATE_KEY

在 https://github.com/settings/apps/ 处你可以找到自己创建并生成的应用，比如我创建的应用名称为`scala-steward-counter2015`

点击进入对应页面就可以查看到 APP_ID



![](https://counter2015.com/picture/github-actions-2-2.png)

这个页面往下翻，可以在 `Private keys` 旁边找到 `Generate a private key`, 点击生成后会下载一个 `.pem` 文件到本地，这就是 APP_PRIVATE_KEY

![](https://counter2015.com/picture/github-actions-2-4.png)



从账户已安装的应用链接：https://github.com/settings/installations/ 中点击可以看到安装的应用，点击对应应用的 `Configure` 按钮可以在地址栏中看到对应的 APP_INSTALLATION_ID

![](https://counter2015.com/picture/github-actions-2-3.png)

有了这三个配置，之后把它们添加到仓库的 secrets 里面就行了

我们可以用一个单独的项目来给账户名下所有的 Scala 项目更新依赖，也可以在项目里面单独配置，这里为了方便演示，选择项目里面单独配置的方式。



在给对应仓库添加好 secrets 后，我们需要编写对应的 workflow 文件，在`.github/workflows/scala-steward.yaml` 中添加如下配置

```yaml
on:
  schedule:
    - cron: '0 0 * * 0'
  workflow_dispatch:

name: Update dependency

jobs:
  scala-steward:
    runs-on: ubuntu-latest
    name: Launch Scala Steward
    steps:
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: 'adopt'
          java-version: '21'

      - name: checkout
        uses: actions/checkout@v4

      - name: Launch Scala Steward
        uses: scala-steward-org/scala-steward-action@v2
        with:
          github-app-id: ${{ secrets.APP_ID }}
          github-app-installation-id: ${{ secrets.APP_INSTALLATION_ID }}
          github-app-key: ${{ secrets.APP_PRIVATE_KEY }}
          github-app-auth-only: true
          github-repository: "counter2015/tccc"
          author-email: "43047562+scala-steward@users.noreply.github.com"
          author-name: "Scala Steward"
```

这个 workflow 的触发方式有手动触发(`workflow_dispatch`) 和定时触发两种。手动触发可以从项目的 Actions  UI 中触发，

定时触发的语法和 cron 类似，你可以通过这个[页面](https://crontab.guru/)查看其含义, 这里`0 0 * * 0` 指的是每周日0点触发（UTC 时间）

你可以手动触发一次看看效果，可能会发现仓库里多了一些依赖更新的 PR



![](https://counter2015.com/picture/github-actions-2-5.png)



对应的徽章

`https://github.com/YOUR_ACCOUNT/YOUR_REPO/actions/workflows/scala-steward.yaml/badge.svg`

```
[![Scala Steward badge](https://img.shields.io/badge/Scala_Steward-helping-blue.svg?style=flat&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAQCAMAAAARSr4IAAAAVFBMVEUAAACHjojlOy5NWlrKzcYRKjGFjIbp293YycuLa3pYY2LSqql4f3pCUFTgSjNodYRmcXUsPD/NTTbjRS+2jomhgnzNc223cGvZS0HaSD0XLjbaSjElhIr+AAAAAXRSTlMAQObYZgAAAHlJREFUCNdNyosOwyAIhWHAQS1Vt7a77/3fcxxdmv0xwmckutAR1nkm4ggbyEcg/wWmlGLDAA3oL50xi6fk5ffZ3E2E3QfZDCcCN2YtbEWZt+Drc6u6rlqv7Uk0LdKqqr5rk2UCRXOk0vmQKGfc94nOJyQjouF9H/wCc9gECEYfONoAAAAASUVORK5CYII=)](https://scala-steward.org)
```



还是以之前的仓库为例，渲染效果为：

[![Scala Steward badge](https://img.shields.io/badge/Scala_Steward-helping-blue.svg?style=flat&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAQCAMAAAARSr4IAAAAVFBMVEUAAACHjojlOy5NWlrKzcYRKjGFjIbp293YycuLa3pYY2LSqql4f3pCUFTgSjNodYRmcXUsPD/NTTbjRS+2jomhgnzNc223cGvZS0HaSD0XLjbaSjElhIr+AAAAAXRSTlMAQObYZgAAAHlJREFUCNdNyosOwyAIhWHAQS1Vt7a77/3fcxxdmv0xwmckutAR1nkm4ggbyEcg/wWmlGLDAA3oL50xi6fk5ffZ3E2E3QfZDCcCN2YtbEWZt+Drc6u6rlqv7Uk0LdKqqr5rk2UCRXOk0vmQKGfc94nOJyQjouF9H/wCc9gECEYfONoAAAAASUVORK5CYII=)](https://scala-steward.org)![Update dependience](https://github.com/counter2015/tccc/actions/workflows/scala-steward.yaml/badge.svg)



文章开头完整的徽章列表如下所示：

```
[![Scala Steward badge](https://img.shields.io/badge/Scala_Steward-helping-blue.svg?style=flat&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAQCAMAAAARSr4IAAAAVFBMVEUAAACHjojlOy5NWlrKzcYRKjGFjIbp293YycuLa3pYY2LSqql4f3pCUFTgSjNodYRmcXUsPD/NTTbjRS+2jomhgnzNc223cGvZS0HaSD0XLjbaSjElhIr+AAAAAXRSTlMAQObYZgAAAHlJREFUCNdNyosOwyAIhWHAQS1Vt7a77/3fcxxdmv0xwmckutAR1nkm4ggbyEcg/wWmlGLDAA3oL50xi6fk5ffZ3E2E3QfZDCcCN2YtbEWZt+Drc6u6rlqv7Uk0LdKqqr5rk2UCRXOk0vmQKGfc94nOJyQjouF9H/wCc9gECEYfONoAAAAASUVORK5CYII=)](https://scala-steward.org)
![Update dependience](https://github.com/counter2015/tccc/actions/workflows/scala-steward.yaml/badge.svg)
[![codecov](https://codecov.io/gh/counter2015/tccc/branch/master/graph/badge.svg)](https://codecov.io/gh/counter2015/tccc)
[![Minimum Scala Version](https://img.shields.io/badge/scala-%3E%3D%203.4.0-8892BF.svg)](https://www.scala-lang.org/)
[![Minimum JDK Version](https://img.shields.io/badge/jdk-%3E%3D%2021-8892BF.svg)](https://openjdk.org/)
```

