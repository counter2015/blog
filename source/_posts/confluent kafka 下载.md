# confluent kafka 下载

title: Confluent Kafka下载
date: 2018-11-25 10:40:59
updated: 2019-01-07 11:50:57
tags: [kafka, confluent] 
categories: [技术]

---
**UPD 2020: 现在看来这篇文章有够水的**

安装前需要满足一定的前提要求，对于生产环境而言，参考[官方文档][1]：
如果是自己学习使用的话，只要不是内存太小和存储空间，应该都没问题。

选用的confluent为5.0.1，其中的kafka版本为2.0.1
```shell
# 下载安装包
$ cd ~/installs
# Notes: `curl -O`命令会把url链接中的文件按名字保存并下载到本地
$ curl -O http://packages.confluent.io/archive/5.0/confluent-oss-5.0.1-2.11.tar.gz
```
下载过程大概是这个样子。

![](https://counter2015.com/picture/cf-0.jpg) 
这大概需要大约10分钟(由于渣网)的时间，正好我们来研究下这个下载链接到底是什么。

我们可以看下这个下载链接，
`http://packages.confluent.io/archive/5.0/confluent-oss-5.0.1-2.11.tar.gz`
`http`是协议名称，<del>吐槽下confluent还没有用上https</del>
`packages.confluent.io/archive/5.0/`中间这一长串链接就别想进去了，都是XML的。
`confluent-oss-5.0.1-2.11.tar.gz`就是我们要下载的包名，oss是open-source software的意思，`5.0.1`代表confluent的版本号，`2.11`代表scala版本为2.11。

好了，现在差不多就下载完成了，接下来是解压缩和打开使用。

```shell
$ tar -zxf confluent-oss-5.0.1-2.11.tar.gz
$ mv confluent-5.0.1 ~/bin
$ cd ~/bin/conflent-5.0.1
$ pwd
/root/bin/confluent-5.0.1
$ ls
bin  etc  lib  README  share  src
```
现在我们已经到了confluent的工作目录下



| 文件夹  | 描述                             |
| ------- | :------------------------------- |
| /bin/   | 用于启动和停止服务的驱动程序脚本 |
| /etc/   | 配置文件                         |
| /lib/   | 系统服务                         |
| /logs/  | 日志文件                         |
| /share/ | jar包和许可证                    |
| /src/   | 依赖于构建平台的源文件           |



至此，confluent的下载就完成了。
参考文档：
[Manual Install using ZIP and TAR Archives][2]


[1]: https://docs.confluent.io/current/installation/system-requirements.html
[2]: https://docs.confluent.io/current/installation/installing_cp/zip-tar.html#prod-kafka-cli-install
