# 使用 InfluxDB作为Prometheus远程存储

title: 使用 InfluxDB作为Prometheus远程存储
date: 2019-08-06 21:59:30
tags: [Prometheus, InfluxDB] 
categories: [技术]

------

## 前言

在[之前的文章](http://counter2015.com/2019/04/30/grafana-moniter/)中我们部署了基于Prometheus和Grafana的监控系统，但是还是有一个不足的地方，监控数据有失效时间，

为了能保留较长时间的历史数据，这里需要引入远程存储

在Prometheus的[集成文档](<https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage>)中列出了不少可以作为存储的组件，从文档数量和安装简易度考虑，这里选择的是influxDB

## InfluxDB 安装配置

以下配置流程适用于centos7

```shell
$ cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF
```

如果你使用的是其他环境可以参考[官方文档](<https://docs.influxdata.com/influxdb/v1.7/introduction/installation/>)

```shell
$ sudo yum install influxdb
$ sudo systemctl start influxdb

#  RFC3339 format (YYYY-MM-DDTHH:MM:SS.nnnnnnnnnZ).
$ influx -precision rfc3339
Connected to http://localhost:8086 version 1.7.7
InfluxDB shell version: 1.7.7
> CREATE DATABASE "prometheus"
> SHOW DATABASES
name: databases
name
----
_internal
prometheus
```

修改prometheus.yml配置文件，添加如下配置，使得监控数据入库
这里没有给influxDB开启认证机制，如果需要暴露给外网，考虑安全性最好配上认证

```shell
$ vim prometheus.yml
remote_write:
  - url: "http://localhost:8086/api/v1/prom/write?db=prometheus"

remote_read:
  - url: "http://localhost:8086/api/v1/prom/read?db=prometheus"
```



修改配置后通过`systemctl`重启prometheus服务

之后在influxDB里就能查看到监控指标

```shell
> show measurements
go_gc_duration_seconds
go_gc_duration_seconds_count
go_gc_duration_seconds_sum
go_goroutines
go_info
go_memstats_alloc_bytes
go_memstats_alloc_bytes_total
go_memstats_buck_hash_sys_bytes
go_memstats_frees_total
go_memstats_gc_cpu_fraction
...
prometheus_tsdb_head_truncations_total
prometheus_tsdb_lowest_timestamp
prometheus_tsdb_lowest_timestamp_seconds
prometheus_tsdb_reloads_failures_total
prometheus_tsdb_reloads_total
...

> select * from go_info
# 输出从略
```

目前只能从最新的监控数据导入，历史数据（Grafana落盘的数据）无法直接导入。

```shell
#当前存储的监控指标有12706个
>  select * from _internal.."database" where "database"=~/prometheus/ order by time desc limit 1
name: database
time                database   hostname              numMeasurements numSeries
----                --------   --------              --------------- ---------
1564985100000000000 prometheus localhost.localdomain 455             12706

```

> Database names, measurements, tag keys, field keys, and tag values are stored only once and always as strings. Only field values and timestamps are stored per-point.
Non-string values require approximately three bytes. String values require variable space as determined by string compression



以上说明来自InfluxDB官网，据此估算每个月需要保存约20亿个点的数据。[不负责任地估算](<https://docs.influxdata.com/influxdb/v1.7/guides/hardware_sizing/>)约要6G的空间



当前最新的稳定开源版本不支持集群功能(可以付费使用商业版或者云版开启集群特性)

这里配置的单机存储就会带来两个问题

- 数据存储的横向扩展
- 集群的可用性

数据存储按之前的计算(20亿个数据6GB/月)，用个大点的SSD，存个几年的数据应该是绰绰有余的

集群的可用性问题可以通过定时备份来解决，流下了贫穷的泪水

## 生产环境的优化

influxDB需要频繁地读写硬盘，因而最好将其运行在1000 IOPS的SSD上。

如果从停机状态恢复，IOPS建议大于2000，否则可能需要较长时间恢复。

默认情况下，表的数据存储在`/var/lib/influxdb/data`目录，wal日志存储在`/var/lib/influxdb/wal`目录下

当磁盘写入负载过高时，将这两个目录挂载到不同的存储设备下，这样可以有效减少磁盘资源的竞用问题。



## 备份和导入

可以参照此[官方文档](<https://docs.influxdata.com/influxdb/v1.7/administration/backup_and_restore/>)

```shell
# 一个简单的示例
$ influxd backup -portable -database <database-name> <path-to-backup>
$ influxd restore -portable -db <database-name> <path-to-backup>
```





## 后续更新

> 数据存储按之前的计算(20亿个数据6GB/月)，用个大点的SSD，存个几年的数据应该是绰绰有余的

这个flag于2020年3月回收，磁盘满了。

观察日志可以发现，首先会报warn,说磁盘空间已满，写不进去了。

再之后preometheus就会自己退出。

其实观察了下，6个月的数据也才40个G，之前的估算是没有问题的，问题在于这个机器上还有其他占用磁盘的东西在跑着。。。

```shell
# 使用这个命令能排查机器上大文件
$ cd / && sudo find . -type f -size +800M  -print0 | xargs -0 du -h

# du -sh /var/lib/influxdb/data/<db name>
# 上述命令能查看具体数据库的存储占用大小
```



手动清理了tsdb的较早的监控数据

```shell
$ influx -precision rfc3339
> USE prometheus
> DELETE WHERE time < '2019-09-01'
```

清理完成后，用`df`命令查看磁盘，发现卡死在`/nfs_shared`的一个疑似远程共享的盘

用`mount`命令查看发现是通过tcp协议分享的其他机器上的磁盘

老老实实联系运维帮忙

之后学习到了一个，大体的解决思路参照[此处](https://www.jianshu.com/p/7e71b5248cb3 )

- `umount`掉有问题的 `/ndf_shared`

- 停止`nfs`和`rpcbind`服务

- 使用排查命令检测问题

  ```shell
  $ cat /etc/mtab
  $ netstat -ntlp
  $ dmesg
  $ crontab -l
  $ cat /etc/issue
  ```

  

- `systemctl restart proc-sys-fs-binfmt_misc.automount`

如果磁盘可用空间实在太小，可以考虑设置过期策略，参见[官方文档](https://docs.influxdata.com/influxdb/v1.7/query_language/database_management/#create-retention-policies-with-create-retention-policy)

下面是一个简单的例子

```shell
> CREATE RETENTION POLICY "800_day_only" ON "prometheus" DURATION 800d REPLICATION 1
```



## 参考链接

1. [Prometheus endpoints support in InfluxDB](<https://docs.influxdata.com/influxdb/v1.7/supported_protocols/prometheus/>)
2. [Howto: Add a new yum repository to install software under CentOS / Redhat Linux](<https://www.cyberciti.biz/tips/rhel5-fedora-core-add-new-yum-repository.html>)
3. [Hardware sizing guidelines](<https://docs.influxdata.com/influxdb/v1.7/guides/hardware_sizing/>)
4. [InfluxDB FAQ](<https://docs.influxdata.com/influxdb/v1.7/troubleshooting/frequently-asked-questions/>)

