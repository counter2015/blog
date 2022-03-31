

# kafka manager简单部署备忘

title: kafka manager简单部署备忘
description:  简单备忘kafka manager 部署过程
date: 2019-3-10 14:55:00
tags: [备忘, kafka] 
categories: [技术]



---





Github上的下载地址：https://github.com/yahoo/kafka-manager/releases

```shell
$ unzip kafka-manager-1.3.3.22.zip 
$ cd kafka-manager-1.3.3.22.zip
# 下载的是源码，所以还需要编译打包成可部署的形式
$ sbt clean dist
$ cp target/universal/kafka-manager-1.3.3.22.zip .
# 虽然解压包的名字一样，但是内容不一样了
$ unzip kafka-manager-1.3.3.22.zip
$ mv kafka-manager-1.3.3.22 kafka-manager
```





## 配置

主要的配置存放在`conf/application.conf`中，最简配置需要配置`zookeeper`集群的服务地址，详细的配置可以参考github主页的README文档。

将配置项`kafka-manager.zkhosts`修改为可用的地址，多个地址之间用逗号分隔。



## 启动

```shell
$ cd kafka-manager
$ bin/kafka-manager -Dconfig.file=conf/application.conf -Dhttp.port=9300
```



启动后需要在UI界面手动添加zookeeper集群信息，完成后的效果如下：

![](https://counter2015.com/picture/kafka-manager-1.jpg) 



![](https://counter2015.com/picture/kafka-manager-2.jpg)





