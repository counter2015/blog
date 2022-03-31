# zookeeper 集群安装部署

title: zookeeper集群安装部署
date: 2018-11-25 10:35:12
tags: zookeeper
categories: [技术]

------

* [zookeeper 集群安装部署](#zookeeper-%E9%9B%86%E7%BE%A4%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2)
  * [系统要求](#%E7%B3%BB%E7%BB%9F%E8%A6%81%E6%B1%82)
  * [下载](#%E4%B8%8B%E8%BD%BD)
  * [\*修改java heap size（可选）](#%E4%BF%AE%E6%94%B9java-heap-size%E5%8F%AF%E9%80%89)
  * [修改配置文件](#%E4%BF%AE%E6%94%B9%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)
  * [启动与验证](#%E5%90%AF%E5%8A%A8%E4%B8%8E%E9%AA%8C%E8%AF%81)
  * [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

## 系统要求

对Zookeeper支持最好的平台是Linux，ZooKeeper在Java中运行，需要推荐使用JDK 8或更高版本。

ZooKeeper集群的最小大小为3，当然你可以部署更多的Zookeeper集群节点，但是最好是奇数个节点，这是为了保证集群选举时能推选出主节点。下面用作演示的版本为3.4.12，可以到<a href="#1">官网</a>下载。


## 下载

首先，选择一台机器下载Zookeeper安装包。
```shell
# 下载安装包
$ cd ~/installs
$ curl -O https://www-us.apache.org/dist/zookeeper/stable/zookeeper-3.4.12.tar.gz
```

> Notes: 如果出现报错信息，curl: (35) Peer reports incompatible or unsupported protocol version.可能是由于curl版本过低，可以将`https`修改为`http`或者使用yum更新curl



```shell
# 解压缩，重命名并移动到对应目录
$ mv zookeeper-3.4.12 zookeeper
$ mv zookeeper ~/bin/
$ cd ~/bin/zookeeper/
```

## *修改java heap size（可选）
由于当内存不足时, swapping操作会严重拖慢zookeeper的性能，所以需要设置Java运行时的"heap size"(堆容量)。

对于内存的分配，还是根据项目和机器情况而定。如果内存够用，适当的大点可以提升zk性能。
```shell
# 新建一个文件
$ vim bin/java.env
```
文件内容如下
```
#!/bin/sh
# heap size MUST be modified according to cluster environment
    
export JVMFLAGS="-Xms512m -Xmx1024m $JVMFLAGS"
```

## 修改配置文件

```shell
# 修改机器的hostname
$ vim /etc/hostname
```
对于三台机器依次修改为`node1`,`node2`,`node3`
修改`/etc/hosts`,添加zookeeper集群对应的域名和IP地址映射
形如`192.168.31.24 node1`

以下操作均在node1上执行
```shell
# 复制示例配置文件
$ cp conf/zoo_sample.cfg conf/zoo.cfg
$ vim conf/zoo.cfg
```

文件内容如下
```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/root/bin/zookeeper/data
dataLogDir=/root/bin/zookeeper/log
clientPort=2181

server.1=node1:2888:3888
server.2=node2:2888:3888
server.3=node3:2888:3888

autopurge.snapRetainCount=3
autopurge.purgeInterval=24
```

| 配置项       | 解释                                                         |
| ------------ | ------------------------------------------------------------ |
| tickTime     | Zookeeper的基本时间度量单位,毫秒                             |
| initLimit    | 节点与leader连接同步的时间，单位为`tickTime`                 |
| syncLimit    | 节点数据同步的时间，单位为`tickTime`                         |
| dataDir      | 数据库快照的存放地址，默认也存放日志（除非配置了`dataLogDir`） |
| dataLogDir   | 存放事务日志的地址，如果和`dataDir`分开有助于提高吞吐        |
| clientPort   | Zookeeper客户端连接的端口地址                                |
| server配置项 | 形如：server.`myid`=`hostname`:`tickport`:`electionport`     |

需要讲解下`server.1=node1:2888:3888`这一行的配置。
`server.`是固定写法，
`1`代表zookeeper节点的ID，唯一的属于[1,255]之间的整数
`node1`为我配置的机器的`hostname`
`2888`为`tickport`, 是zookeeper节点用于内部通信的端口
`3888`为`electionport`,是zookeeper节点用于选举的端口

`autopurge.snapRetainCount` 和 `autopurge.purgeInterval` 设置于清除最近3天的快照外的其他快照.

接下来需要创建`dataLogDir`和`dataDir`文件夹
```shell
$ cd /root/bin/zookeeper
$ mkdir data
$ mkdir log
```

将文件分发到各个节点
```shell
$ cd /root/bin
$ scp -r zookeeper node2:$PWD
$ scp -r zookeeper node3:$PWD
```

在集群的三个节点分别设置`myid`，以node2为例
```shell
$ cd ~/bin/zookeeper/data
$ echo "2" > myid
```

在集群的三个节点配置环境变量
```shell
vim ~/.bashrc
```
添加如下几行
```
# Zookeeper
export ZOOKEEPER_HOME=$LOCAL/zookeeper
export PATH=$ZK_HOME/bin:$PATH
```
```shell
# 使配置生效
source ~/.bashrc
```


## 启动与验证
在集群上分别执行启动命令
```shell
$ zkServer.sh start
$ jps
16276 QuorumPeerMain
```
可以看到有`QuorumPeerMain`进程，说明zookeeper正常启动
```shell
# 查看各个节点的状态
$ zkServer.sh statuss
```
> Notes: 如果启动出错，可以通过zookeeper.out的日志排查错误，常见的错误是端口被禁用或占用，这时候需要修改配置文件中对应的端口号,还可能是Hosts文件没有正确配置，或者防火墙禁用了端口

不同的节点返回的状态不同，比如
```shell
# node1
[root@node1 etc]# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /root/bin/zookeeper/bin/../conf/zoo.cfg
Mode: follower

# node2
[root@node2 ~]# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /root/bin/zookeeper/bin/../conf/zoo.cfg
Mode: follower

# node3
[root@node3 ~]# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /root/bin/zookeeper/bin/../conf/zoo.cfg
Mode: leader

```
出现如上信息，说明我们的zookeeper集群成功启动。
















## 参考资料

1. [Zookeeper学习之路（二）集群搭建][1] 
2. [ZooKeeper Getting Started Guide][2]
3. [ZooKeeper Administrator's Guide][3]
4. [<a name="1">Zookeeper下载</a>][4]
5. [ZooKeeper: 简介, 配置及运维指南][5]


[1]: http://www.cnblogs.com/qingyunzong/p/8619184.html
[2]: https://zookeeper.apache.org/doc/current/zookeeperStarted.html#sc_RunningReplicatedZooKeeper
[3]: https://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_supportedPlatforms
[4]: http://zookeeper.apache.org/releases.html
[5]: https://www.cnblogs.com/neooelric/p/9230967.html