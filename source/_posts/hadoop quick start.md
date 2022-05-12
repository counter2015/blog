

# hadoop快速安装指南

title: hadoop快速安装指南
date: 2018-12-15 16:32:02
tags: hadoop
categories: [技术]



---

## 系统环境

- CentOS 7
- Hadoop版本选用2.9.1<del>Hadoop 3.x都出来了(#`O′)</del>
- 安装方法适用于hadoop 2.7.1
- 为了方便后续的测试和开发，选择使用[伪分布式配置](https://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/SingleCluster.html#Pseudo-Distributed_Operation)
- 需要先安装好JDK 8



## 下载和安装

```shell
$ cd ~/installs
$ curl -O https://archive.apache.org/dist/hadoop/common/hadoop-2.9.1/hadoop-2.9.1.tar.gz
$ tar -zxf hadoop-2.9.1.tar.gz 
$ mv hadoop-2.9.1 hadoop
$ mv hadoop ~/bin
$ cd ~/bin/hadoop
```



## 配置

```shell
$ vim ~/.bashrc
```



一个简单的示例如下，个人习惯将安装包放在`~/installs`，各种应用存放至`~/bin`,用户配置修改`~/.bashrc`,所以这个配置文件的存放前缀为`/root/bin`<del>为了避免各种烦人的权限问题直接上root</del>。也可以根据个人的喜好自行设置。

```shell
export LOCAL=/root/bin

# Hadoop
export HADOOP_HOME=$LOCAL/hadoop
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

# Java
# for case that other application depended on `JAVA_HOME`, this part should be add to the head of `PATH`
export JAVA_HOME=$LOCAL/jdk
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

修改配置文件后，**切记让配置生效**

```shell
$ source ~/.bashrc
```



修改主机名和hosts文件

```shell
$ echo "mycentos">/etc/hostsname
$ echo "mycentos 192.168.22.129"
```

其中，修改主机名后可能需要重启后才能生效，生效后的shell命令前缀如`[root@mycentos hadoop]# `

ip地址需要自己查询并修改，这里`192.168.22.129`是这台机器的ip。



接下来是修改hadoop的配置，我们主要需要修改以下两个文件`core-site.xml`和`hdfs-site.xml`,这个两个文件位置在/root/bin/hadoop/etc/hadoop目录下

```xml
# core-site.xml
<configuration>
    <property>
        <!--- 存放hadoop数据文件的位置 -->
        <name>hadoop.tmp.dir</name>
        <value>file:/root/bin/hadoop/tmp</value>
        <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <!--- 文件系统的端口，从其他机器访问需要配置域名解析 -->
        <name>fs.defaultFS</name>
        <value>hdfs://mycentos:9000</value>
    </property>
</configuration>

# hdfs-site,xml
<configuration>
    <property>
        <!--- 文件副本数量，生产环境一般设置为3 -->
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <!--- NameNode文件存放路径 -->
        <name>dfs.namenode.name.dir</name>
        <value>file:/root/bin/hadoop/tmp/dfs/name</value>
    </property>
    <property>
        <!--- DataNode文件存放路径 -->
        <name>dfs.datanode.data.dir</name>
        <value>file:/root/bin/hadoop/tmp/dfs/data</value>
    </property>
</configuration>


```



## 启动

启动前需要对NameNode进行格式化

> Notes: 非首次格式化时，可能会造成DataNode和NameNode的Cluster ID不一致的问题，这会导致DataNode无法正常启动，需要手动修改/root/bin/hadoop/tmp/dfs/data/current目录下的VERSION文件中的clusterID为/root/bin/hadoop/tmp/dfs/name/current目录下VERSION文件中的对应值。

```shell
$ hdfs namenode -format
```



使用启动脚本启动

```shell
$ start-dfs.sh
$ jps
```

若成功启动`jps`命令下会显示 “NameNode”、”DataNode”和"SecondaryNameNode"。启动不成功时可以通过查看日志排查问题。启动日志存放在/root/bin/hadoop/logs下。

```shell
# 创建一个文件夹验证一下
$ hdfs dfs -mkdir /tmp
$ hdfs dfs -ls /
```

