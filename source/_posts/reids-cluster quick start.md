# Redis集群快速搭建指南


title: Redis集群快速搭建指南
date: 2019-01-13 20:28:22
tags: redis
categories: [技术]

---



> **WARNING**  文章的部分内容可能已经**严重**过时，请酌情食用。

## 快速开始

这里以3.2.1版本为例
下载地址:http://download.redis.io/releases/

```shell
$ cd ~/installs
$ wget http://download.redis.io/releases/redis-3.2.1.tar.gz
$ tar -zxf redis-3.2.1.tar.gz
$ cd redis-3.2.1
$ make
```

编译完成后，我们只需要以下几个文件就能启动一个redis客户端


- 启动服务器所需的`src/redis-server`
- redis命令行`src/redis-cli`
- 配置文件`redis.conf`
- 集群配置工具`redis-trib.rb`

将这三个文件复制到同一个目录下
```shell
$ mkdir ~/bin/redis-3.2.1
$ cp src/redis-server \
     src/redis-cli \
     src/redis-trib.rb \
     redis.conf ~/bin/redis-3.2.1/
$ cd ~/bin/redis-3.2.1/
```

快速示例
```shell
$ ./redis-server
$ ./redis-cli
127.0.0.1:6379> set 1 1
OK
127.0.0.1:6379> get 1
"1"
```

以上形式启动的redis有几个问题:

- redis服务运行在前台
- 外部程序无法访问到redis

## 单机配置
对于第一个问题,将配置文件做如下修改
```
# 设置启动方式为后台启动
daemonize yes
```

对于第二个问题
```
# 注释掉该行
# bind 127.0.0.1
# 关闭保护模式
protected-mode no
```

数据持久化AOF开启，关于AOF的详细解释可以查看文末的参考资料
```
appendonly yes
```

其他优化，移动了一些文件的位置
```
pidfile redis.pid
logfile redis.log
```

安全性，禁用部分高危命令
```
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
```

至此，单机模式的配置文件修改完成

## 集群配置

redis3和4都能用类似方法安装，redis5开始使用`redis-cli`安装集群,具体可以参照文末的参考资料。

一个较简洁的Redis集群配置
```
port 7000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 15000
appendonly yes
daemonize yes
```
需要注意的是，最小的Redis集群需要配置3个主节点，3个从节点。

为此，需要创建一个新目录，并以端口命名创建以下目录。
```shell
$ mkdir cluster-test
$ cd cluster-test
$ mkdir 7000 7001 7002 7003 7004 7005
```

给每个文件夹添加配置文件，内容就是之前那简单的几行，注意修改`port`。
```shell
$ cd 7000
$ vim redis.conf
```

将之前我们编译好的`redis-server`复制到相应位置
```shell
$ cd ..
$ cp ../redis-server .
```

在每个端口启动redis,以7000为例，这里可能需要开启多个控制台窗口
```shell
$ cd 7000
$ ../redis-server ./redis.conf
```

现在我们将所有的redis实例都启动起来了，还需要配置些信息才能把它们搞成集群

还记得我们之前复制的`redis-trib.rb`吗？现在就是用它的时候了。

`rb`文件是用Ruby编写的，所以需要先安装Ruby
直接用yum安装并简单验证下
```shell
$ cp ../redis-trib.rb .
$ yum install ruby
$ ruby -v
ruby 2.0.0p648 (2015-12-16) [x86_64-linux]
$ gem
RubyGems is a sophisticated package manager for Ruby.  This is a
basic help message containing pointers to more information.

  Usage:
    gem -h/--help
    gem -v/--version
    gem command [arguments...] [options...]

  Examples:
    gem install rake
    gem list --local
    gem build package.gemspec
    gem help install

  Further help:
    gem help commands            list all 'gem' commands
    gem help examples            show some examples of usage
    gem help platforms           show information about platforms
    gem help <COMMAND>           show help on COMMAND
                                   (e.g. 'gem help install')
    gem server                   present a web page at
                                 http://localhost:8808/
                                 with info about installed gems
  Further information:
    http://guides.rubygems.org
```
为了能正常运行`redis-trib.rb`,需要安装redis在ruby上的支持程序
```shell
$ gem install redis -v 3.2.1
```
使用`redis-trib.rb`创建集群
```shell
$ ./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \
> 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
127.0.0.1:7000
127.0.0.1:7001
127.0.0.1:7002
Adding replica 127.0.0.1:7003 to 127.0.0.1:7000
Adding replica 127.0.0.1:7004 to 127.0.0.1:7001
Adding replica 127.0.0.1:7005 to 127.0.0.1:7002
M: 174387047806fa01aa40f21d81e483c6750d46a9 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: 30e6454bc49df7972aa0bb3622afc1de18a9cc62 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: 83bfb46ec9d65d075b4dad0090e7b872526bc6d2 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
S: 293680aab55d0cec9e73f527f16efeea9bee552c 127.0.0.1:7003
   replicates 174387047806fa01aa40f21d81e483c6750d46a9
S: df83916424dd4b810b41187d24422745de2ee5ab 127.0.0.1:7004
   replicates 30e6454bc49df7972aa0bb3622afc1de18a9cc62
S: 5a11ad3086b3d43ddbed4628459138c7b01ad2eb 127.0.0.1:7005
   replicates 83bfb46ec9d65d075b4dad0090e7b872526bc6d2
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join....
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: 174387047806fa01aa40f21d81e483c6750d46a9 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: 30e6454bc49df7972aa0bb3622afc1de18a9cc62 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: 83bfb46ec9d65d075b4dad0090e7b872526bc6d2 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
M: 293680aab55d0cec9e73f527f16efeea9bee552c 127.0.0.1:7003
   slots: (0 slots) master
   replicates 174387047806fa01aa40f21d81e483c6750d46a9
M: df83916424dd4b810b41187d24422745de2ee5ab 127.0.0.1:7004
   slots: (0 slots) master
   replicates 30e6454bc49df7972aa0bb3622afc1de18a9cc62
M: 5a11ad3086b3d43ddbed4628459138c7b01ad2eb 127.0.0.1:7005
   slots: (0 slots) master
   replicates 83bfb46ec9d65d075b4dad0090e7b872526bc6d2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

至此，集群的搭建工作就基本完成了，可以根据自己的需要修改配置文件。

<del>**鬼知道为啥hexo把配置文件里的英文标点转换成了中文标点了，食用时注意替换**</del>

 <a href="https://counter2015.com/2019/01/18/bug1">{% spoiler 问题已解决 %}</a>

<details>
  <summary>端口为7000节点的完整配置文件示例如下(去除注释): </summary>
  <p></p>
  <pre><code>
protected-mode no
port 7000
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile redis.pid
loglevel notice
logfile redis.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir ./
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 15000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
  </code>  </pre>
</details>
运行时某个端口的目录如下
```shell
$ cd 7000
$ ls
appendonly.aof  dump.rdb  nodes-7000.conf  redis.conf  redis.log  redis.pid
```

|文件名|说明|
|---------|-|
|appendonly.aof|AOF记录文件，记录redis的每次操作，用于持久化|
|dump.rdb|Redis快照文件，用于持久化|
|nodes-7000.conf|节点的状态文件，自动生成，无需改动|
|redis.conf|Redis配置文件|
|redis.log|Redis日志文件|
|redis.pid|记录Redis示例运行的pid|

集群的简单测试

```shell
$ cd redis-3.2.1/cluster-test
$ ls
7000  7001  7002  7003  7004  7005  redis-cli  redis-server  redis-trib.rb
$ ./redis-cli -c -p 7000
127.0.0.1:7000> set a 1
-> Redirected to slot [15495] located at 127.0.0.1:7002
OK
127.0.0.1:7002> set b 2
-> Redirected to slot [3300] located at 127.0.0.1:7000
OK
127.0.0.1:7000> get a
-> Redirected to slot [15495] located at 127.0.0.1:7002
"1"
127.0.0.1:7002> get b
-> Redirected to slot [3300] located at 127.0.0.1:7000
"2"
127.0.0.1:7000> get b
"2"
127.0.0.1:7000> 


```





## 参考资料

1. [官方文档 cluster-tutorial](https://redis.io/topics/cluster-tutorial)
2. [官方文档 persistence](https://redis.io/topics/persistence)
3. [官方文档 security](https://redis.io/topics/security)
4. [官方文档 download](https://redis.io/download)
5. [Redis-使用redis-trib构建集群](https://blog.csdn.net/qq_35946990/article/details/78957618)
