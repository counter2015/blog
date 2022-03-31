# Redis Live部署备忘

title: Redis Live部署备忘
description:  简单备忘Redis Live部署过程
date: 2019-3-10 19:54:44
tags: [备忘, redis] 
categories: [技术]





---



github项目地址：https://github.com/nkrode/RedisLive

## 安装

```shell
# 从github上下载
$ wget https://github.com/kumarnitin/RedisLive/zipball/master
$ unzip master
$ mv nkrode-RedisLive-e1d7763 RedisLive
$ cd RedisLive
$ cd src
$ cp redis-live.conf.example redis-live.conf
```

### 依赖

- tornado
- redis.py
- python-dateutil

为了方便起见，使用Python的隔离环境工具[vitualenv](https://virtualenv.pypa.io/en/latest/)来下载对应安装包

```shell
# 如果没有安装virtualenv，先安装它
$ pip install --user virtualenv

# 创建一个干净的环境
$ virtualenv --no-site-packages env1
$ source env1/bin/activate

# 依次安装依赖
$ pip install -r ../requirements.txt 
```





## 配置

- 编辑redis-live.conf：

- 配置`RedisServers`为要监视的redis实例。

- 需要有个数据库来存储“Redis-live”的信息，这里选择的是sqlite,将`DataStoreType` 设置为 `sqlite`，配置存储路径即可。

一个示例配置文件

```json
{
	"RedisServers":
	[ 
		{
  			"server": "1.2.3.4",
  			"port" : 6379
		}
	],

	"DataStoreType" : "sqlite",
	
	"SqliteStatsStore" :
	{
		"path":  "db/redislive.sqlite"
	}
}

```

目前不支持一个ip配置多个端口，需要配置的话 另(M)起(D)一(Z)行(Z)

  

## 运行

运行脚本`start.sh`如下

```shell
$ source env1/bin/activate
$ nohup ./redis-monitor.py --duration=120 >> log/redis-monitor.log 2>&1 &
$ nohup ./redis-live.py >> log/redis-live.log 2>&1 &	
```

这个只是个示例，实际运行过程中redis的`monitor`命令占用的资源不能忽视，因而要设置时间，这个例子设置的时间是120秒，可以使用定时任务按照自己的需要调整调用`redis-monitor.py`的频率。



你可以在 `http://localhost:8888/index.html` 查看监控页面了，默认运行端口为`8888`



![](https://counter2015.com/picture/redis-live-1.jpg)

  