# Prometheus + Grafana 监控配置指北：打造日志监控系统

title: Prometheus + Grafana 监控配置指北：打造日志监控系统
date: 2019-04-30 21:59:30
tags: [Prometheus, Grafana] 
categories: [技术, 搭积木]

------
2022-05-12 更新: 
**本文写于2019年，现在更建议通过 k8s operator 的形式来进行安装部署**


本文不再维护，请查看[新版](https://counter2015.com/2020/04/13/grafana-monitor-2/)

## 前置知识

![](https://counter2015.com/picture/grafana-1.jpg)



▲ 图片来源：<https://prometheus.io/docs/introduction/overview/>

给上图各部分做个简单的介绍

- Prometheus Server: Prometheus服务端，由于存储及收集数据，提供相关api对外查询用。
- Exporter: 类似传统意义上的被监控端的agent，有区别的是，它不会主动推送监控数据到server端，而是等待server端定时来收集数据，即所谓的主动监控。
- Pushagateway: 用于网络不可直达而居于exporter与server端的中转站。
- Alertmanager: 报警组件
- Web UI: Prometheus的web接口，可用于简单可视化，及语句执行或者服务状态监控。
- PromQL: 类似于SQL(其实并不是SQL)的查询语言，用于获取计算Prometheus的监控数据
- Grafana: 替换Prometheus Web UI做数据可视化的工作



下面会先用Prometheus + NodeExporter + Grafana 构建一套对机器基本性能的监控服务

| 机器名称 | host    | ip          | 运行程序            |
| -------- | ------- | ----------- | ------------------- |
| A        | monitor | 192.168.1.1 | Grafana、Prometheus |
| B        | node1   | 192.168.1.2 | NodeExporter        |
| C        | node2   | 192.168.1.3 | NodeExporter        |
| D        | node3   | 192.168.1.4 | NodeExporter        |
| E        | node4   | 192.168.1.5 | Mtail               |

下文为了方便说明是在哪台机器上操作，使用A/B/C/D来指代。

做完这一步后，预期效果如下

![](https://counter2015.com/picture/grafana-2.jpg)

监控报警

![](https://counter2015.com/picture/grafana-3.jpg)

<del>自定义Exporter实现事务TPS统计</del>

后来发现了Mtail，直接写写DSL就行了



## 准备工作

### Node Exporter

在需要监控的机器(B、C、D)上**分别**安装启动Node Exporter

```plain
# 下载Node Exporter
$ cd ~/installs
$ wget https://github.com/prometheus/node_exporter/releases/download/v0.17.0/node_exporter-0.17.0.linux-amd64.tar.gz
$ tar -zxf node_exporter-0.17.0.linux-amd64.tar.gz 

# 将文件夹移动至合适的位置（个人习惯）
$ mv node_exporter-0.17.0.linux-amd64 ~/bin
$ cd ~/bin/node_exporter-0.17.0.linux-amd64/

# 后台运行Node Exporter,默认端口地址为9100
$ nohup ./node_exporter &
```

### Prometheus

[下载页](https://prometheus.io/download/)

```plain
$ cd ~/installs
$ wget https://github.com/prometheus/prometheus/releases/download/v2.9.1/prometheus-2.9.1.linux-amd64.tar.gz
$ tar -zxf prometheus-2.9.1.linux-amd64.tar.gz 
$ mv prometheus-2.9.1.linux-amd64 ~/bin
$ cd ~/binprometheus-2.9.1.linux-amd64/

```

修改配置文件`prometheus.yml`为如下内容,注意缩进

```yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
      labels:
        instance: prometheus
  - job_name: 'job_1'
    static_configs:
    - targets: ['192.168.1.2:9100']
      labels:
        instance: instance1
    - targets: ['192.168.1.3:9100']
      labels:
        instance: instance2
    - targets: ['192.168.1.4:9100']
      labels:
        instance: instance3

```



配置完成后,启动Prometheus

```plain
$ nohup ./prometheus --config.file=prometheus.yml &
```

此时服务能在192.168.1.1:9090通过浏览器访问。

![](https://counter2015.com/picture/grafana-4.jpg)


我们可以看到，各个服务都正常启动了。

![](https://counter2015.com/picture/grafana-5.jpg)

### Grafana

[下载安装](<https://grafana.com/grafana/download?platform=linux>)

```plain
$ wget https://dl.grafana.com/oss/release/grafana-6.1.3-1.x86_64.rpm 
$ sudo yum localinstall grafana-6.1.3-1.x86_64.rpm 

# 安装饼状图的插件，之后有个Dashboard需要依赖此插件
$ grafana-cli plugins install grafana-piechart-panel

# 启动服务
$ systemctl daemon-reload
$ systemctl start grafana-server
$ systemctl status grafana-server

## 设置开机启动
$ sudo systemctl enable grafana-server.service
```





如果你希望能邮件邀请别人注册Grafana，需要对配置文件`/etc/grafana/grafana.ini`的部分配置项做如下修改

```ini
#################################### SMTP / Emailing ##########################
[smtp]
enabled = true
host = localhost:25
;user =
# If the password contains # or ; you have to wrap it with trippel quotes. Ex """#password;"""
;password =
;cert_file =
;key_file =
skip_verify = false
from_address = admin@grafana.localhost
;from_name = Grafana
# EHLO identity in SMTP dialog (defaults to instance_name)
;ehlo_identity = dashboard.example.com

[emails]
welcome_email_on_sign_up = true

#################################### Distributed tracing ############
[tracing.jaeger]
# Enable by setting the address sending traces to jaeger (ex localhost:6831)
;address = localhost:6831
# Tag that will always be included in when creating new spans. ex (tag1:value1,tag2:value2)
;domain = localhost

# Redirect to correct domain if host header does not match domain
# Prevents DNS rebinding attacks
;enforce_domain = false

# The full public facing url you use in browser, used for redirects and emails
# If you use reverse proxy and sub path specify full url (with sub path)
root_url = http://192.168.1.1:3000

```
此时服务能在192.168.1.1:3000被访问

> NOTE: 修改配置文件后，需要重启grafana服务才能生效。

初次登录，账户名和密码都是`admin`

登录后会提示修改密码，进入后的主界面类似于下图

![](https://counter2015.com/picture/grafana-6.jpg)



接下来需要配置`Data Source`和`Dashboard`

![](https://counter2015.com/picture/grafana-7.gif)



现在配置好了数据来源，接下来就需要一个仪表盘来展示收集的数据，这里用的是一个预设的模板

[*1 Node Exporter 0.16 0.17 for Prometheus 监控展示看板*](<https://grafana.com/dashboards/8919>)

模板id为8919，直接可以导入。

![](https://counter2015.com/picture/grafana-8.jpg)





配置DashBoard选项

![](https://counter2015.com/picture/grafana-9.png)





完成效果

![](https://counter2015.com/picture/grafana-10.jpg)



## 获取监控数据

### 自定义实现Exporters

有时候需要的监控指标官方的Exporters没有实现，这时候就需要自己动手了。

在官方的[Client Libraries](<https://prometheus.io/docs/instrumenting/clientlibs/>)里支持

- Go
- Java or Scala
- Python
- Ruby

这里选择用Go语言来实现，原因如下

- Prometheus是go编写的，所以社区中使用go的博文较多，可以方便参考
- go编写的可以编译成可执行文件，不需要在部署时额外安装编译器

首先我们来创建一个最简Exporters，之后在它上面逐渐添加我们需要监控的指标

首先配置一下项目结构

```plain
$ mkdir -p ~/dev/PrometheusExporter
$ cd ~/dev/PrometheusExporter/
$ export GOPATH=$PWD
$ mkdir src
$ cd src
$ mkdir app
$ cd ..
$ tree
tree
.
`-- src
    `-- app


```

接下来的主要代码就放在`src/app`里。


```plain
$ cd src/app
$ vim main.go
```
```go
package main

import (
  "net/http"

  log "github.com/Sirupsen/logrus"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
  //This section will start the HTTP server and expose
  //any metrics on the /metrics endpoint.
  http.Handle("/metrics", promhttp.Handler())
  log.Info("Beginning to serve on port :8080")
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

这里有几个依赖库，需要`go get`先下载下来。

```plain
$ go get -v
github.com/Sirupsen/logrus (download)
Fetching https://golang.org/x/sys/unix?go-get=1
https fetch failed: Get https://golang.org/x/sys/unix?go-get=1: dial tcp 216.239.37.1:443: connect: connection refused
package golang.org/x/sys/unix: unrecognized import path "golang.org/x/sys/unix" (https fetch: Get https://golang.org/x/sys/unix?go-get=1: dial tcp 216.239.37.1:443: connect: connection refused)
github.com/prometheus/client_golang (download)
github.com/beorn7/perks (download)
github.com/golang/protobuf (download)
github.com/prometheus/client_model (download)
github.com/prometheus/common (download)
github.com/matttproud/golang_protobuf_extensions (download)
github.com/prometheus/procfs (download)

```

可以看到，(由于众所周知的原因)有一个包下载失败了。

这个问题的解决办法有很多，但是有很多也不那么好用，这里用的是手动下载[github](<https://github.com/golang/sys>)上的代码。

```plain
$ git clone https://github.com/golang/sys.git $GOPATH/src/golang.org/x/sys
$ go get -v
golang.org/x/sys/unix
github.com/Sirupsen/logrus
github.com/beorn7/perks/quantile
github.com/golang/protobuf/proto
github.com/prometheus/client_model/go
github.com/prometheus/client_golang/prometheus/internal
github.com/matttproud/golang_protobuf_extensions/pbutil
github.com/prometheus/common/internal/bitbucket.org/ww/goautoneg
github.com/prometheus/common/model
github.com/prometheus/common/expfmt
github.com/prometheus/procfs
github.com/prometheus/client_golang/prometheus
github.com/prometheus/client_golang/prometheus/promhttp
app

```

现在我们可以编译运行了

```plain
$ go run main.go
INFO[0000] Beginning to serve on port :8080   
```

可以通过浏览器，或者另起一个终端观察运行结果

```plain
$ curl localhost:8080/metrics
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 6
...
...
...
# HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.
# TYPE promhttp_metric_handler_requests_total counter
promhttp_metric_handler_requests_total{code="200"} 0
promhttp_metric_handler_requests_total{code="500"} 0
promhttp_metric_handler_requests_total{code="503"} 0

```

```plain
$ vim collect.go
```

```go
package main

import (
	"github.com/prometheus/client_golang/prometheus"
)

//Define the metrics we wish to expose
var fooMetric = prometheus.NewGauge(prometheus.GaugeOpts{
	Name: "foo_metric", Help: "Shows whether a foo has occurred in our cluster"})

var barMetric = prometheus.NewGauge(prometheus.GaugeOpts{
	Name: "bar_metric", Help: "Shows whether a bar has occurred in our cluster"})

func init() {
	//Register metrics with prometheus
	prometheus.MustRegister(fooMetric)
	prometheus.MustRegister(barMetric)

	//Set fooMetric to 1
	fooMetric.Set(0)

	//Set barMetric to 0
	barMetric.Set(1)
}
```

这里要注意下，`init`这个函数是有点`magic`的，这个函数会自动在`main`函数运行前执行，在`init`函数内，我们简单地注册了两个类型为`Gauge`的监控指标，并且给他们设置了初始值。

```plain
$ go run main.go collector.go
```

```plain
$ curl localhost:8080/metrics
# HELP bar_metric Shows whether a bar has occurred in our cluster
# TYPE bar_metric gauge
bar_metric 1
# HELP foo_metric Shows whether a foo has occurred in our cluster
# TYPE foo_metric gauge
foo_metric 0
...
...
...
```



### mtail

mtail是一个能帮助我们方便地提取日志字段到监控指标的工具，

其使用的正则遵循[RE2-style regular expression syntax](<https://github.com/google/re2/wiki/Syntax>)的标准，但是也受到Go语言正则实现方式的限制。

```plain
# 下载二进制文件
$ cd ~/installs
$ wget https://github.com/google/mtail/releases/download/v3.0.0-rc29/mtail_v3.0.0-rc29_linux_amd64

# 准备测试环境和数据
$ mkdir -p ~/dev/mtail
$ cp mtail_v3.0.0-rc29_linux_amd64 mtail
$ mv matil ~/dev/mtail
$ cd ~/dev/mtail
$ echo -e "line 1\na\nb\n" >> test.log
$ cat linecounter.mtail
# ~/linecounter.mtail
# simple line counter
counter line_count
/$/ {
  line_count++
}

# 开始测试
$ ./mtail --progs linecounter.mtail --logs test.log



# 另启动一个窗口，观察结果
$ cd ~/dev/mtail
$ curl http://localhost:3903/metrics | grep line_count
# HELP line_count defined at linecounter.mtail:3:9-18
# TYPE line_count counter
line_count{prog="linecounter.mtail"} 0
# HELP mtail_line_count number of lines received by the program loader
# TYPE mtail_line_count untyped

# 当往日志中新加入文本中时
$ echo "test1" >> test.log
$ curl http://localhost:3903/metrics | grep line_count
# HELP line_count defined at linecounter.mtail:3:9-18
# TYPE line_count counter
line_count{prog="linecounter.mtail"} 1
# HELP mtail_line_count number of lines received by the program loader
# TYPE mtail_line_count untyped
mtail_line_count 1

```



如果你想写一些比较复杂的过滤规则或者正则，可以先去mtail的[github][10]上查阅相关文档，

测试正则推荐直接用这个[网站](<https://www.regexpal.com/>)

一个简单的mtail DSL示例如下

```plain
counter count_lines
gauge total_someapp_response_time
/response header/{
  total_someapp_response_time = 0
  count_lines++
  /time (\d+)/ {
    total_someapp_response_time += $1
  }
}

```

对应的日志可以为

```plain
INFO 2019-04-28 17:00:00 ok, response header, used time 100 ms
```

修改prometheus.yml的对应端口

```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
      labels:
        instance: prometheus
  - job_name: 'job_1'
    static_configs:
    - targets: ['192.168.1.2:9100']
      labels:
        instance: instance1
    - targets: ['192.168.1.3:9100']
      labels:
        instance: instance2
    - targets: ['192.168.1.4:9100']
      labels:
        instance: instance3
  - job_name: 'logmoniter'
    static_configs:
    - targets: ['192.168.1.5:3903']
      labels:
        instance: app1

```





随便新建个仪表盘，里面加个图

![](https://counter2015.com/picture/grafana-11.jpg)

## 监控报警

比较通用的做法就是邮件报警了，首先添加一个新的邮件报警频道

![](https://counter2015.com/picture/grafana-12.jpg)

然后设置该频道内接收邮件的邮箱地址

![](https://counter2015.com/picture/grafana-13.jpg)

配置好后，我们可以测试下

![](https://counter2015.com/picture/grafana-14.jpg)

如果不能发送邮件，请检查之前是否有正确配置`/etc/grafana/grafana.ini`中的`smtp`是否打开，同时确保grafana运行的机器能使用smtp邮件服务端口25。

接下来可以对某个具体的表格，设置邮件报警的规则和检查频率

![](https://counter2015.com/picture/grafana-15.jpg)

![](https://counter2015.com/picture/grafana-17.jpg)

收到的邮件大概长这个样子

![](https://counter2015.com/picture/grafana-16.jpg)

至此，基本的操作流程就跑完了，如果你想定制自己的监控指标/图表，都能在这个框架上通过简单的修改快速得到结果。

## 运行时存储

prometheus将监控指标存放至其可执行文件下的`data`文件夹内，包括write-ahead-log (WAL) 日志和以[tsdb](<https://github.com/prometheus/tsdb/tree/master/docs/format>)格式压缩的监控数据。

平均而言，Prometheus每个样本仅使用大约1-2个字节。在自己搭建测试环境中，

理论上估计100G硬盘空间能支持300个节点以15秒间隔，15天的监控数据持久化。

当前运行监控有80个节点，15秒采集间隔，15天数据失效时间，prometheus的监控数据占用总磁盘空间为2GB.

grafana的数据默认以sqlite的格式存储至本地名为`grafana.db`的文件中。



## What's More

[使用Scala开发应用程序 Prometheus Exporter](<http://counter2015.com/2019/05/17/scala-exporter/>)

[使用 InfluxDB作为Prometheus远程存储](<http://counter2015.com/2019/08/06/influx-storage/>)



## 参考资料

1. [Grafana: Getting Start][1]
2. [Grafana: Docs][2]
3. [Grafana: Github][3]
4. [Grafana: DashBoards][4]
5. [Prometheus: Getting Start][5]
6. [Prometheus: Docs][6]
7. [Prometheus: Exporters][7]
8. [Prometheus: Client Java][8]
9. [Prometheus: Client Go][9]
10. [Mtail: Github][10]
11. [Telegraf: Github][11]
12. [Prometheus metrics / OpenMetrics code instrumentation][12]
13. [A Noob's Guide to Custom Prometheus Exporters (Revamped!)][13]
14. [A practical intro to Prometheus][14]
15. [Monitoring log files using some metrics exporter + Prometheus + Grafana][15]
16. [Prometheus官网文档中编译random示例时go语言报错的解决办法][16]
17. [使用Prometheus+grafana打造高逼格监控平台][17]
18. [prometheus + alertmanager + grafana强强联合][18]





[1]: <https://grafana.com/docs/guides/getting_started/>
[2]: <https://grafana.com/docs/>
[3]: <https://github.com/grafana/grafana>
[4]: <https://grafana.com/dashboards>
[5]: https://prometheus.io/docs/prometheus/latest/getting_started/
[6]: <https://prometheus.io/docs/>
[7]: <https://prometheus.io/docs/instrumenting/exporters/>
[8]: <https://github.com/prometheus/client_java>
[9]: <https://github.com/prometheus/client_golang>
[10]: <https://github.com/google/mtail>
[11]: <https://github.com/influxdata/telegraf>
[12]: <https://sysdig.com/blog/prometheus-metrics/>
[13]: <https://rsmitty.github.io/Prometheus-Exporters-Revamp/>
[14]: http://groob.io/posts/prometheus-intro/
[15]:  <https://stackoverflow.com/questions/41160883/monitoring-log-files-using-some-metrics-exporter-prometheus-grafana>
[16]: https://blog.csdn.net/aolia2000/article/details/84900890
[17]: <https://blog.51cto.com/youerning/2050543?from=timeline>
[18]: https://blog.51cto.com/13053917/2339969










