# hbase安装指南

title: hbase安装指南
date: 2019-03-23 11:20:02
tags: [hadoop,hbase]
categories: [技术]



---


## 依赖

- JDK 8
- Hadoop 2.7.1,[下载地址](https://archive.apache.org/dist/hadoop/common/)
- Zookeeper 3.4.12

请参照[适配表格](http://hbase.apache.org/book.html#hadoop)选择合适的Hadoop/HBase版本，否则很可能出现奇怪的bug。

![](https://counter2015.com/picture/hbase-1.jpg) 

**本文假设环境[zookeeper集群安装部署](http://counter2015.com/2018/11/28/zookeeper%20%E9%9B%86%E7%BE%A4%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2/), [hadoop快速安装指南](http://counter2015.com/2018/12/13/hadoop%20quick%20start/)安装按照对应文章步骤,且已经配置好java,hadoop,zookeeper的环境变量**。

## 安装

官网地址：http://hbase.apache.org/

使用的HBase版本为2.1.3，安装方式为[伪分布式](http://hbase.apache.org/book.html#pseudo)<del>因为我没钱租服务器</del>

```shell
$ tar -zxf hbase-2.1.3-bin.tar.gz 
$ mv hbase-2.1.3 hbase
$ mv hbase ../bin/
$ cd ../bin/hbase/
$ pwd
/root/bin/hbase
```



## 配置

```shell
$ vim conf/hbase-env.sh
# The java implementation to use.  Java 1.8+ required.
# 这里的jdk路径按照个人设置来配置
export JAVA_HOME=/root/bin/jdk
```



```shell
$ vim conf/hbase-site.xml
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://mycentos:8020/hbase</value>
  </property>
  
</configuration>
```



```shell
$ bin/hbase shell
hbase(main):001:0> 
```



至此，单机的hbase部署完成。

配置环境变量

```shell
export LOCAL=/root/bin

# Hbase
export HBASE_HOME=$LOCAL/hbase
export PATH=$HBASE_HOME/bin:$PATH
```



退出后，修改配置文件来安装伪分布式Hbase

```shell
$ vim conf/hbase-env.sh
# 使用外部zookeeper而不是hbase自带的
export HBASE_MANAGES_ZK=false
```

修改`hbase-site.xml`，完整的配置文件如下

```xml
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://mycentos:9000/hbase</value>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.tmp.dir</name>
    <value>/root/bin/hbase/tmp</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/root/bin/zookeeper/data</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>mycentos</value>
  </property>
</configuration>
```

- `hbase.rootdir`的前半部分地址和hadoop配置文件`core-site.xml`中的`fs.defaultFS`相同，这个例子中，`fs.defaultFS`配置的是`hdfs://mycentos:9000`，该目录用来用来持久化HBase的数据
- `hbase.cluster.distributed`需要设置为`true`
- `hbase.zookeeper.quorum`，多个域名之间用逗号分隔，这里我只在本地启动了一个zookeeper
- `hbase.zookeeper.property.dataDir`,和zookeeper配置文件`zoo.cfg`中的`dataDir`配置的路径相同

## 启动

```shell
$ start-hbase.sh
```



## 验证

```shell
# 出现HMaster,HRegionServer说明启动成功
$ jps
86529 DataNode
87393 NodeManager
107136 HMaster
86324 NameNode
86807 SecondaryNameNode
12730 SupportedKafka
11931 QuorumPeerMain
107277 HRegionServer
87260 ResourceManager
109422 Jps

$ hbase shell
# hbase的命名空间默认是default
hbase(main):001:0> list_namespace
NAMESPACE         
default                              
hbase                      
2 row(s)
Took 1.0839 seconds  

# 新建一个名为test的命名空间
hbase(main):003:0> create_namespace 'test'
Took 0.4517 seconds

# 在test的命名空间内新建一个表test_table_1,其有一个列族名为column_family_1
hbase(main):005:0> create 'test:test_table_1', 'column_family_1'
Created table test:test_table_1
Took 1.5447 seconds                                                             
=> Hbase::Table - test:test_table_1

#   往test:test_table_1表中行键为rowkey1的位置，列族为column_family_1,列为column_1
# 的位置插入value1
hbase(main):006:0> put 'test:test_table_1', 'rowkey1', 'column_family_1:column_1', 'value1'
Took 0.2476 seconds  

# 查看test:test_table_1的所有值
hbase(main):007:0> scan 'test:test_table_1'
ROW                                          COLUMN+CELL              
rowkey1       column=column_family_1:column_1,timestamp=1553147678290,value=value1     
1 row(s)
Took 0.0630 seconds 

# 查看test:test_table_1的表的属性
hbase(main):010:0> describe 'test:test_table_1'
Table test:test_table_1 is ENABLED
test:test_table_1 
COLUMN FAMILIES DESCRIPTION
{NAME => 'column_family_1', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', KEEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE => 'false',DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', MIN_VERSIONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WRITE => 'false', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 'false', COMPRESSION => 'NONE', BLOCKCACHE => 'true', BLOCKSIZE => '65536'}
1 row(s)
Took 0.5791 seconds  
```



HBase web监控界面 默认端口号为16010

![](https://counter2015.com/picture/hbase-2.png) 



## 参考资料

1. [HBase官方文档][1] 
2. [HBase默认配置文件][2]
3. [HBase学习之路 （二）HBase集群安装][3]
4. [HBase 伪分布式搭建（使用外部ZK)][4]
5. [常见错误排查][5]


[1]: http://hbase.apache.org/2.1/book.html
[2]: https://github.com/apache/hbase/blob/master/hbase-common/src/main/resources/hbase-default.xml
[3]: https://www.cnblogs.com/qingyunzong/p/8668880.html
[4]: https://blog.csdn.net/HG_Harvey/article/details/79594552
[5]: https://github.com/apache/hbase/blob/master/src/main/asciidoc/_chapters/troubleshooting.adoc#trouble.master.startup