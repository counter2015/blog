# Akka Projection Cassandra 拆分迁移踩坑记录

title: Akka Projection Cassandra 拆分迁移踩坑记录
date: 2023-02-03 20:20:24
tags: [Cassandra, k8s, akka]
categories: [技术]

---


## 背景介绍

之前介绍过一次[如何在不停机的情况下水平拓展并迁移数据中心](https://counter2015.com/2022/05/14/cassandra-migrate/) ，从而给数据库节点更换更大的数据。

现在又有了新的需求，由于业务变更，需要提供按租户垂直拆分的数据，将整套服务按照租户隔离重新部署。

之前所有的数据都是按照事件变更的形势，通过 protobuf 编码存储到 cassandra 集群内部的，数据表的 schema 使用的是 akka persitence cassandra 自带的，按照 `persistence_id` 作为标识，存放不同实体的变更事件，整体架构是 CQRS(Command and Query Responsibility Segregation) 的形式, 下游通过消费变更事件，触发对应的流水线和模型训练服务。

![](https://counter2015.com/picture/cqrs-1.png)


实现过程中，租户对应的信息是以编码的形式存放到 `persistence_id` 中，具体的实现方式是

`<url_encoded_tenant>:<domain_entity_name>:<entity_id>`

比如
`%E6%B5%8B%E8%AF%95:doc:1` 标识的实体为 `测试`租户中类型为`doc`，实体id为`1`的实体



Cassandra 为了实现分布式的存储，在查询时对应做出了权衡，没有完全指定分区键(**Partition Key**, 负责数据在集群内的分布)，或者错误指定聚集键(**Clustering Key**, 负责数据在每一个分区上的排序)的情况下是不能使用 `where` 查询条件的



同时，Cassandra 不支持直接进行 `like "%:test:%"`的查询，因为这会导致查询全部数据节点扫全表的操作，会极大影响性能。

虽然可以通过手动开启并建立 [SASI（SSTable Attached Secondary Index） 索引](https://cassandra.apache.org/doc/latest/cassandra/cql/SASI.html) 并启用  `Contains` 模式来支持以上查询，但这会引入新的问题

- 对于租户这一种级别较宽的分类条件，返回的数据过多，扫描代价过高
- SASI 的实施细节中提到，此功能会影响 MemTable 刷新的生命周期，这会极大地影响写入、压实、修复等操作的吞吐量

- 数据行存在删除语义时，过多的墓碑会严重影响 SASI 索引的查询效率



因此，有必要找出其他方案来对数据进行租户级别的垂直拆分。

## 前期调研



### DataX

其实本来我是不想调研这个方案的，奈何有云玩家发言说：“这个东西我看阿里不就做过吗，你拿过来改一改就行，很简单的。”



我对阿里的开源组件用得比较少，同时也是因为 fastjson 还有 ant-design 所以对他们出品的东西不太敢用



> "让你解析一段 json, 你咋弹出了一个计算器呢？"
>
> fastjosn: "你别管，你就说快不快吧。"



[Datax](https://github.com/alibaba/DataX) 是阿里云提供的一个数据集成组件，提供了多种数据源的导入、导出功能。

进去项目地址后一看，更新还是最近的，但仔细一看 cassandra 组件相关的[提交历史](https://github.com/alibaba/DataX/commits/master/cassandrareader)，这部分从2019年10月首次提交后基本就没咋更新了，最近的几个 commit 也只是不痛不痒的代码格式化和小修小补，依赖版本更新啥的。



把项目拉下来跑了下试试，猜了几个配置终于猜对了，但是没办法拆分过滤操作，而且跑到一半有概率报错，首先排除一个错误答案。



### DataCenter sync

其实之前也提到过，可以通过数据中心同步 / 离线 Snapshot 的方式来做，不过这样需要在迁移完成后，删除掉所有不属于这个租户的数据，这个机器和运维成本太高，所以放弃了。





### Spark Connector



Cassandra 的商业化公司 Datastax 提供了一个 [spark-cassandra-connector](https://github.com/datastax/spark-cassandra-connector) 的库，可以方便地使用 spark 来计算 casandra 数据, 但是考虑到一下几个问题，所以没有选用

- 生产使用的 cassandra 版本是 4.x，而这个库的兼容性表格只支持到 3.x 版本，可能会有兼容问题
- 之前并未搭建 hadoop + spark 环境，需要重新搭建，成本较高
- 数据量不大(TB级别)，使用 spark 有点大材小用





考虑了半天，发现这个需求还是自己开发会比较方便些





## 需求梳理

我们的目标是开发一个离线任务，对于单租户的导入和导出，运行时间控制在8小时内，这样可以在凌晨时开启任务，最大程度减少由于用户写入操作引发的数据不一致问题。

在 cassandra 数据导入完成后，可以通过重放事件来恢复对应实体的最新状态，同时往下游消息队列写入变更事件，触发其他服务的流水线操作。



akka projection / persitence  本身没有提供迁移的工具，所以这里只能靠自己来 hack

首先，需要确定需要导出哪些表的数据





![](https://counter2015.github.io/picture/akka-tables.png)





可以看到，这里有三个不同的 keyspace: `akka`, `akka_projection` 以及 `akka_snapshot`，由于 akka 对应的文档并没有详细解释每张表的含义以及设计思路，所以这里我是通过表的命名以及数据来猜测对应的用途。

- all_persitence_ids 存放实体id列表
- messages 存放消息
- metadata 这张表是空的，不用迁移
- tag_scanning  看起来像是记录了实体id当前最后的事件序列号
- tag_views 这张表里面有很多数据列和 `messages`表差不多，不过对应的分区键和聚集键有所不同，看起来像是为了适应按照时间查询的形式，额外定义的表
- tag_write_progress 记录了实体维度下，对应分类标签当前已经处理到的时间偏移量(timeuuid offset), 最后的序列号以及 当前已经处理到的进度

- offset_store 记录了不同的 projection(比如负责写入到 postgres 的，负责写入到 kafka 的)纬度下，不同分区下最后一次写入的时间，以及已经当前处理到的位置
- projection_management 这张表是空的，不用迁移
- snapshots 存放实体状态的中间计算结果，方便快速从较多的事件中计算恢复



这里面，数据最大的部分就是事件，在 cassandra 中的存储方式是十六进制字符串。

考虑到调试方便，这里选用 json 作为导出文件的组织形式，事件相关的数据采用 base64  编码



导出时，拉取所有实体的id，过滤出对应租户的数据，然后把这部分数据



导出过程中需要有地方存放数据，可以先使用本地/容器内 磁盘，然后上传到 s3 的形式。

导入是从 s3 下载对应数据，再将其流式地插入到新数据库中，避免内存压力。



这种做法基本上原样保留了所有业务数据，但是还是会丢失一些 cassandra 层面上的元信息，比如如果有设置数据 TTL(time to live)的情况下，重新插入会修改对应列的 `writetime`。如果新旧集群的配置不一样，可能数据的分布也会有所区别。



不过还好，这部分影响不大。



调试过程中，为了缓解用户（也就是我）的等待焦虑，加入了一个进度条，用来显示当前表的处理进度。不过这个需要额外扫全表来确认数据总条数，耗费的时间更多。



## 开发与排错



### Cassandra 查询超时

查询时出现超时，发现是卡在 count 的时候，quill 生成 query 时，流式获取数据，默认情况下每一批次的量是5000条，读取的超时时间是5s。

虽然可以通过服务端修改超时时间的方式绕开这个问题，但会带来不必要的副作用，所以这里手动减小了每一批获取数据的条数（这里手动跑了多次 benchtest 取了个较优的值）。

同时在统计 count 数量时，没有必要获取所有字段，只需要拿 `persistence_id` 字段就可以满足过滤和统计，减小网络流量的开销。



### 数据一致性问题

数据导出后，发现有部分消息丢失了中间的变更记录，从数据库查询后，发现是因为任务在导出时一致性级别没设置好，把 cassandra java driver 中的 `consistency` 设置成`LOCAL_QUORUM`,避免遗漏数据。



### ZStream OOM

数据处理使用的是 [ZIO](https://github.com/zio/zio) 的流式 API `ZStream`，测试过程中发现有内存泄漏的情况，于是提交了 [issue](https://github.com/zio/zio/issues/7535)

没想到6个小时内就有项目成员提供反馈了，第二天就修好了，速度贼快，不得不夸一夸 ZIO 社区的反馈速度。

但是由于还没发版，我就用其他方式绕开这个坑了。

顺便在使用  zio-s3 的时候发现[没有正确返回状态码](https://github.com/zio/zio-s3/issues/365)，本来想提交个 patch 混个  pr, 结果由于质量太低，成抛砖引玉的，人家直接自己修了。



### 资源不足

导入过程中，由于集群内能分配到的资源有限（32 core 64G 机器上面跑了100多个服务），经常出现拉取 s3 数据超时的情况，这个就只能加上重试的逻辑，然后调高超时时间来缓解了, 对于比较大的文件，就不使用直接消费的形式了——因为处理时间过长，一个网络波动就挂了——而是直接先下载到本地文件，再慢慢消费。



资源不足还会引发另外一个问题，本来是准备通过事件回放的形式来恢复数据的，但是回放时会吧 cpu 打得很高，导致事件计算频繁超时。

这里首先想到的是加上一个限速器，以信号量的方式来限制并发，但是可能是由于姿势不太对，由于 akka projection 没有暴露底层 akka stream 消费事件数据

的限流参数，即使设置同一时间只能处理一个实体，cpu 的消耗依然能打到2。



在资源有限的情况下看起来实在是没法满足这个需求，只能采取折衷的方案，将 postgres 里面存放的数据状态也编写一份导入导出的任务执行， 这种情况下，kafka 里存放的消息也没有太大必要存放了，下游其他服务如果需要维护状态，需要自己迁移这部分数据而不是通过消费数据来解决，这样可以减少数据拆分的整体执行时间。



### 处理时间过长

在测试过程中，发现对于1000万条数据，导入导出一共需要花费8小时。于是把之前的进度条功能移除了，减少扫描全表的时间。

同时由于 postgres readside 那边可以直接过去到对应租户下的实体数据，可以直接走索引查询，把从 cassandra 查询实体 id 这一步修改成从 postgres 里面查询实体 id。

同时，上面的表没必要都进行迁移，[部分表的数据](https://doc.akka.io/docs/akka-persistence-cassandra/current/events-by-tag.html#events-by-tag-reconciliation)可以等到实际调用时再进行计算。

经过上面的一系列优化，可以把时间从8小时降低到3小时。



### 单个文件过大，无法上传到 s3

为了保证文件上传失败后可以分块重试，我使用的是 mutipart upload API, 而这一个 API 会限制分块的最大数量，以及每一个分块的大小，这个和对应的云存储厂商有关。

[AWS](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html) 支持最大10,000 个分块，每个分块大小为 5M ~ 5G, 单个文件最大为5TB

[腾讯云 COS](https://cloud.tencent.com/document/product/436/14112) 支持最大10,000 个分块，每个分块大小为1M ~ 5G, 并没有特别说明支持的最大单个文件大小，理论上来说按照计算应该是 50TB

而我使用的 zio-s3 默认分块大小为 5M, 所以超过 50G的文件就处理不动了，把这个参数改了下就修好了。



## 可以改进的地方



- 导出数据时，文件体积可以进一步降低，首先可以通过更换编码：入 cbor-json, protobuf 或其他的二进制格式。上传数据前，可以先进行压缩操作, 减少网络传输的时间。

- 对于较大的文件数据，可以按数据行进行拆分成多个小文件，这样上传下载的时候可以并发。同时也有利于设置检查点。





## 参考链接




1. [Stackoverflow: Cassandra 中 partition key, composite key 和 clustering key 的区别][1]
2. [Akka persistence cassandra 文档][2]





[1]: https://stackoverflow.com/questions/24949676/difference-between-partition-key-composite-key-and-clustering-key-in-cassandra

[2]: https://doc.akka.io/docs/akka-persistence-cassandra/current/index.html

