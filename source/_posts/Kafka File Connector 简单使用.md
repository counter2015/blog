# Kafka File Connector 简单使用

title: Kafka File Connector 简单使用
date: 2018-12-2 11:56:29
tags: [kafka] 
categories: [技术]

---




File Connector作用简单明了，从文件将数据导入kafka的topic中,以及,从kafka的topic中将数据导出到文件。

> Notes: 如果只是为了测试和学习，可以直接参考confluent官网的教程，使用`confluent start`命令。生产环境下不推荐使用CLI启动confluent。

本文依赖于confluent 5.0.1，如果尚未下载，可以参照我之前写的[安装][4]和[启动][5]。
```shell
$ jps
85043 QuorumPeerMain
6204 ConnectDistributed
193805 SupportedKafka
```
需要先启动以上三个服务。



大多数配置都依赖于connector，因此这里不能都提到。但是，有一些公用的配置：

- name - connector的唯一名称。尝试再次使用相同名称注册将失败。
- connector.class - connecor的Java类
- tasks.max - 应为此connector创建的最大任务数。如果connector无法达到此级别的并行性，则可能会创建更少的任务。
- key.converter - （可选）覆盖worker设置的默认转换器。
- value.converter - （可选）覆盖worker设置的默认转换器。

以下示例未开启`Schema Registry`，并在`etc/kafka/connect-distributed.properties`中设置了如下配置项:

- key.converter=org.apache.kafka.connect.json.JsonConverter
- value.converter=org.apache.kafka.connect.json.JsonConverter
- key.converter.schemas.enable=false
- value.converter.schemas.enable=false

## Source Connector


构造测试数据
```shell
$ cd /root/bin/confluent
$ for i in {1..3}; do echo "log line $i"; done >> test.txt
```


file connector 配置文件`source.json`
```
{
  "name": "file-source-1.0",
  "config": {
    "connector.class": "FileStreamSource",
    "tasks.max": "1",
    "file": "/root/bin/confluent/test.txt",
    "topic": "connect-test-1.0"
  }
}
```

> Notes: 默认情况下topic是自动创建的，但是如果需要做详细配置最好手动创建


通过REST API上传配置文件
```shell
$ curl -X POST -H "Content-Type: application/json" --data @source.json http://node1:8083/connectors
{"name":"file-source-1.0","config":{"connector.class":"FileStreamSource","tasks.max":"1","file":"/root/bin/confluent/test.txt","topic":"connect-test-1.0","name":"file-source-1.0"},"tasks":[],"type":null}

# 查看是否上传成功
$ curl node1:8083/connectors
["file-source-1.0"]
```

此时我们的connector就会自动检测文件的变化并上传到`file-source-1.0`的topic里了，现在我们来查看下结果

```shell
# 启动控制台的消费者查看topic内的消息
$ bin/kafka-console-consumer --bootstrap-server node1:9092 --topic connect-test-1.0 --from-beginning
"log line 1"
"log line 2"
"log line 3"
```

如果另外打开一个终端，并在`test.txt`的末尾添加新的数据，也会被传入到`file-source-1.0`中。

## Sink Connector

配置文件`sink.json`
```
{
  "name": "file-sink-1.0",
  "config": {
    "connector.class": "FileStreamSink",
    "tasks.max": "1",
    "file": "/root/bin/confluent/test.sink.txt",
    "topics": "connect-test-1.0"
  }
}
```
> Notes:`source.json`中配置项为`topic`，而这里是`topics`

通过REST API上传配置文件
```shell
$ curl -X POST -H "Content-Type: application/json" --data @sink.json http://node1:8083/connectors
{"name":"file-sink-1.0","config":{"connector.class":"FileStreamSink","tasks.max":"1","file":"/root/bin/confluent/test.sink.txt","topics":"connect-test-1.0","name":"file-sink-1.0"},"tasks":[],"type":null}

# 查看是否上传成功
$ curl node1:8083/connectors
["file-source-1.0","file-sink-1.0"]
```
当前目录下会有一个文件`test.sink.txt`，内容和`test.txt`一致。
```shell
# 如果配置错了，可以删除connector
curl -X DELETE  http://node1:8083/connectors/file-sink-1.0
```
## 参考资料
1.[Use Kafka Connect to import/export data][1]
2.[Tutorial: Moving Data In and Out of Kafka][2]
3.[Running Kafka Connect][3]


[1]: https://kafka.apache.org/quickstart#quickstart_kafkaconnect
[2]: https://docs.confluent.io/current/connect/quickstart.html
[3]: https://kafka.apache.org/documentation/#connect_running
[4]: https://counter2015.github.io/2018/11/27/confluent%20kafka%20%E4%B8%8B%E8%BD%BD/
[5]: https://counter2015.github.io/2018/12/04/confluent%20kafka%20%E9%9B%86%E7%BE%A4%E9%85%8D%E7%BD%AE%E5%90%AF%E5%8A%A8/
