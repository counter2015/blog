# Kafka HDFS Sink Connector 简单使用

title: Kafka HDFS Sink Connector 简单使用
date: 2018-12-15 19:18:33
tags: [kafka, hadoop] 
categories: [技术]



---


## 介绍

HDFS Connector允许以各种格式将数据从Kafka topic 导出到HDFS文件。

## 依赖
本文依赖于[Confluent][3]，[Hadoop][4]。
```shell
$ jps
85043 QuorumPeerMain
6204 ConnectDistributed
193805 SupportedKafka

# 切换到工作目录
$ cd /root/bin/confluent
```
需要先启动以上三个服务。

```shell
# 测试环境下可以通过Confluent Cli来开发
$ confluent start

# 生产环境下，需要根据自己的需要调整启动参数，以下是几个例子

# 启动zookeeper
$ zkServer.sh start

# 启动kafka
$ ${CONFLUENT_HOME}/bin/kafka-server-start \
-daemon ${CONFLUENT_HOME}/etc/kafka/server.properties

# 启动connect
KAFKA_HEAP_OPTS="-Xms1G -Xmx8G" 
KAFKA_LOG4J_OPTS="-Dlog4j.configuration=file:${CONFLUENT_HOME}/etc/kafka/connect-log4j.properties"  
connect-distributed -daemon  $CONFLUENT_HOME/etc/kafka/connect-distributed.properties

```



在启动Confluent Platform之前，需要先确保Hadoop在本地或远程运行，并且知晓HDFS URL,并在/etc/hosts中配置域名解析,`core-site.xml`中的`fs.default`,本文中为`hdfs://mycentos:9000`

## 准备数据

简单对kafka进行基本操作举几个例子
```
# 列出当前所有的topic
$ bin/kafka-topics --list --zookeeper node1:2181

# 创建一个topic
$ bin/kafka-topics --create --zookeeper node1:2181 --topic test-1 --partitions 1 --replication-factor 1

# 删除一个topic
$ bin/kafka-topics --delete --zookeeper node1:2181 --topic test-1 
```

> **WARNING**: topic命名虽然说基本没有什么限制，但是有几点需要注意，合法字符是数字和大小写字母加上下划线`_`,减号`-`,点`.`，topic的长度最好不超过200，因为最大的长度也就240左右。`.`和`_`不能同时使用，这会导致很多奇怪的问题，下面举个例子。
```
# 创建topic `test-1`
$ bin/kafka-topics --create --zookeeper node1:2181 --topic test_1 --partitions 1 --replication-factor 1

WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Created topic "test_1".


# 我们尝试建立 `test.1`的topic
$ bin/kafka-topics --create --zookeeper node1:2181 --topic test.1 --partitions 1 --replication-factor 1

WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Error while executing topic command : Topic 'test.1' collides with existing topics: test_1
[2018-12-04 18:31:42,730] ERROR org.apache.kafka.common.errors.InvalidTopicException: Topic 'hdfs_test.1' collides with existing topics: hdfs_test_1
 (kafka.admin.TopicCommand$)
```

构造测试数据


```shell
$ bin/kafka-topics --create --zookeeper node1:2181 --topic test-1 --partitions 1 --replication-factor 1
Created topic "test-1".

# 通过控制台随便传入些json格式的数据,最后按Ctrl+C退出
$ bin/kafka-console-producer --broker-list node1:9092 --topic test-1
>{"type":"type1", "time":"2332", "msg":"sadf 12124"}
>{"type":"type2", "time":"2333", "msg":"sadf 12324"}
>{"type":"type1", "time":"2333", "msg":"sadf 123124"}
>{"type":"type3", "time":"2334", "msg":"sadf 1231"}


# 可以在输入的同时另外打开一个终端观察是否数据正常传入
$ bin/kafka-console-consumer --bootstrap-server node1:9092 --topic test-1 --from-beginning
{"type":"type1", "time":"2332", "msg":"sadf 12124"}
{"type":"type2", "time":"2333", "msg":"sadf 12324"}
{"type":"type1", "time":"2333", "msg":"sadf 123124"}
{"type":"type3", "time":"2334", "msg":"sadf 1231"}
```
万事具备，只欠connector的配置了。

## 配置

配置文件`hdfs-1.json`
```json
{
  "name": "hdfs-sink-1",
  "config": {
    "connector.class": "io.confluent.connect.hdfs.HdfsSinkConnector",
    "tasks.max": "1",
    "topics": "test-1",
    "hdfs.url": "hdfs://mycentos:9000/tmp/confluent/test1",
    "flush.size": "1",
    "format.class": "io.confluent.connect.hdfs.json.JsonFormat" 
  }
}

```

> Notes: 当前用户需要有在`hdfs.url`的权限,并实现在hdfs上创建对应目录

通过REST API上传配置文件
```shell
$ curl -X POST -H "Content-Type: application/json" --data @hdfs-1.json http://node1:8083/connectors
{"name":"hdfs-sink-1","config":{"connector.class":"io.confluent.connect.hdfs.HdfsSinkConnector","tasks.max":"1","topics":"test-1","hdfs.url":"hdfs://h28227:9000/tmp/confluent/test1","flush.size":"1","format.class":"io.confluent.connect.hdfs.json.JsonFormat","name":"hdfs-sink-1"},"tasks":[],"type":null}
```



## 检验

可以在hdfs对应目录查看到结果
默认情况下，hdfs的存放将topics和logs分开存放在`hdfs.url`下的对应目录下
```shell
$ hdfs dfs -ls /tmp/confluent/test1
Found 2 items
drwxr-xr-x   - root supergroup          0 2018-12-06 16:08 /tmp/confluent/test1/logs
drwxr-xr-x   - root supergroup          0 2018-12-06 16:08 /tmp/confluent/test1/topics
```

我们可以看到数据存放到`test-1`目录下，而临时文件存放到`+tmp`目录下
```shell
$ hdfs dfs -ls /tmp/confluent/test1/topics
Found 3 items
drwxr-xr-x   - root supergroup          0 2018-12-06 16:08 /tmp/confluent/test1/topics/+tmp
drwxr-xr-x   - root supergroup          0 2018-12-06 16:08 /tmp/confluent/test1/topics/test-1
```

可以看到，这里分了三个区存
```shell
$ hdfs dfs -ls /tmp/confluent/test1/topics/test-1
Found 3 items
drwxr-xr-x   - root supergroup          0 2018-12-06 16:08 /tmp/confluent/test1/topics/test-1/partition=0
drwxr-xr-x   - root supergroup          0 2018-12-06 16:08 /tmp/confluent/test1/topics/test-1/partition=1
drwxr-xr-x   - root supergroup          0 2018-12-06 16:08 /tmp/confluent/test1/topics/test-1/partition=2
```

这和`test-1`的分区信息有关
```shell
$ bin/kafka-topics --zookeeper node1:2181 --describe --topic test-1
Topic:test-1	PartitionCount:3	ReplicationFactor:1	Configs:
	Topic: test-1	Partition: 0	Leader: 1004	Replicas: 1004	Isr: 1004
	Topic: test-1	Partition: 1	Leader: 1005	Replicas: 1005	Isr: 1005
	Topic: test-1	Partition: 2	Leader: 1001	Replicas: 1001	Isr: 1001
```

下载验证hdfs上的文件
```shell
$ hdfs dfs -getmerge /tmp/confluent/test1/topics/test-1/* 1

$ cat 1
{"msg":"sadf 12124","time":"2332","type":"type1"}
{"msg":"sadf 123124","time":"2333","type":"type1"}
{"msg":"sadf 1231","time":"2334","type":"type3"}
{"msg":"sadf 12324","time":"2333","type":"type2"}

$ hdfs dfs -ls /tmp/confluent/test1/topics/test-1/partition=0 
Found 1 items
-rw-r--r--   3 root supergroup         50 2018-12-06 16:08 /tmp/confluent/test1/topics/test-1/partition=0/test-1+0+0000000000+0000000000.json

$ hdfs dfs -cat /tmp/confluent/test1/topics/test-1/partition=0/*
{"msg":"sadf 12124","time":"2332","type":"type1"}

$ hdfs dfs -cat /tmp/confluent/test1/topics/test-1/partition=1/*
{"msg":"sadf 123124","time":"2333","type":"type1"}
{"msg":"sadf 1231","time":"2334","type":"type3"}

$ hdfs dfs -cat /tmp/confluent/test1/topics/test-1/partition=2/*
{"msg":"sadf 12324","time":"2333","type":"type2"}

```
## 使用更多的参数


默认情况下，存储位置是按照`partition`来生成路径的，像之前的例子，存放数据的位置为`hdfs://<hdfs.url>/topics/<topic-name>/partition=<i>/`,可以不可以按数据的时间来存放呢？

### 按时间存放

新建一个`t3`,用`kafka-console-consumer`传入下面数据并查看

```shell
$ bin/kafka-console-consumer --bootstrap-server node1:9092 --from-beginning --topic t3
{"f1":"type1", "f2":"2333", "f3":"sadf 123124"}
{"f1":"type3", "f2":"2334", "f3":"sadf 1231"}
{"f1":"type1", "f2":"2332", "f3":"sadf 12124"}
{"f1":"type2", "f2":"2333", "f3":"sadf 12324"}
```





配置文件`hdfs-t1.json`
```json
{
  "name": "hdfs-sink-t1",
  "config": {
    "connector.class": "io.confluent.connect.hdfs.HdfsSinkConnector",
    "tasks.max": "1",
    "topics": "t3",
    "hdfs.url": "hdfs://mycentos:9000/tmp/confluent/test3",
    "flush.size": "1",
    "format.class": "io.confluent.connect.hdfs.json.JsonFormat",
    "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "path.format": "YYYY-MM-dd",
    "partition.duration.ms": "100",
    "locale": "CHINA",
    "timezone": "Asia/Shanghai"
  }
}
```


上传配置文件
```shell
$ curl -X POST -H "Content-Type: application/json" --data @hdfs-t1.json http://node1:8083/connectors
```

查看hdfs文件
```
$ hdfs dfs -ls /tmp/confluent/test3/topics/t3
Found 1 items
drwxr-xr-x   - root supergroup          0 2018-12-07 12:01 /tmp/confluent/test3/topics/t3/2018-12-07

$ hdfs dfs -ls /tmp/confluent/test3/topics/t3/2018-12-07
Found 4 items
-rw-r--r--   3 root supergroup         45 2018-12-07 12:01 /tmp/confluent/test3/topics/t3/2018-12-07/t3+0+0000000000+0000000000.json
-rw-r--r--   3 root supergroup         46 2018-12-07 12:01 /tmp/confluent/test3/topics/t3/2018-12-07/t3+1+0000000000+0000000000.json
-rw-r--r--   3 root supergroup         44 2018-12-07 12:01 /tmp/confluent/test3/topics/t3/2018-12-07/t3+1+0000000001+0000000001.json
-rw-r--r--   3 root supergroup         45 2018-12-07 12:01 /tmp/confluent/test3/topics/t3/2018-12-07/t3+2+0000000000+0000000000.json
```



### 按多个字段归类存放

配置文件`hdfs-f2.json`

```json
{
  "name": "hdfs-sink-f2",
  "config": {
    "connector.class": "io.confluent.connect.hdfs.HdfsSinkConnector",
    "tasks.max": "1",
    "topics": "t1",
    "hdfs.url": "hdfs://mycentos:9000/f2",
    "flush.size": "1",
    "format.class": "io.confluent.connect.hdfs.json.JsonFormat",
    "partitioner.class": "io.confluent.connect.storage.partitioner.FieldPartitioner",
    "partition.field.name": "f1,f2"
  }
}

```



提取字段需要传入的数据类型为avro,所以需要启动`schema-registry`服务，并通过`kafka-avro-console-producer`来创建数据。

如果是通过`confluent start`命令，该服务会在本节点自动启动，否则需要手动启动

```shell
# 如果是confluent start会启动该服务，否则需要手动启动
${CONFLUENT_HOME}/bin/schema-registry-start -daemon ${CONFLUENT_HOME}/etc/schema-registry/schema-registry.properties
```
配置文件schem-registry.properties需要做如下修改
```
# 不同的节点设置不同的域名，如node2就设置为http://node2:8081
listeners=http://node1:8081

kafkastore.connection.url=node1:2181,node2:2181,node3:2181
```

每个节点都需要分别配置并启动。



准备数据

```shell
$ bin/kafka-avro-console-producer --broker-list node1:9092 --topic t1 \
--property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"},{"name":"f2","type":"string"},{"name":"f3","type":"string"}]}'

>{"f1":"type1","f2":"2332","f3":"sadf 12124"}
>{"f1":"type1","f2":"2333","f3":"sadf 123124"}
>{"f1":"type2","f2":"2333","f3":"sadf 12324"}
>{"f1":"type3","f2":"2334","f3":"sadf 1231"}

```



上传hdfs-connector 配置文件

```shell
$ curl -X POST -H "Content-Type: application/json" --data @hdfs-f2.json http://node1:8083/connectors
```



查看hdfs文件

```shell
$ hdfs dfs -ls /f2/topics/t1
Found 3 items
drwxr-xr-x   - root supergroup          0 2018-12-10 12:12 /f2/topics/t1/f1=type1
drwxr-xr-x   - root supergroup          0 2018-12-10 12:12 /f2/topics/t1/f1=type2
drwxr-xr-x   - root supergroup          0 2018-12-10 12:12 /f2/topics/t1/f1=type3

$hdfs dfs -ls /f2/topics/t1/f1=type1
Found 2 items
drwxr-xr-x   - root supergroup          0 2018-12-10 12:12 /f2/topics/t1/f1=type1/f2=2332
drwxr-xr-x   - root supergroup          0 2018-12-10 12:12 /f2/topics/t1/f1=type1/f2=2333

```



> 提问：存放的格式为`/f1=type1/f2=2332`,有没有办法改成`/type1/2332`呢？
>
> UPD:<https://github.com/confluentinc/kafka-connect-hdfs/issues/397>
>
> 太长不看：暂时不考虑支持这种做法

## 参考资料

1. [Kafka Connect HDFS][1]
2. [HDFS Connector Configuration Options][2]


[1]: https://docs.confluent.io/current/connect/kafka-connect-hdfs/index.html
[2]: https://docs.confluent.io/current/connect/kafka-connect-hdfs/configuration_options.html#hdfs
[3]: https://counter2015.github.io/2018/12/04/confluent%20kafka%20%E9%9B%86%E7%BE%A4%E9%85%8D%E7%BD%AE%E5%90%AF%E5%8A%A8/
[4]:https://counter2015.github.io/2018/12/13/hadoop%20quick%20start/
