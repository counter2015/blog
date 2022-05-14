# 记一次Cassandra 在 k8s 环境下的数据中心迁移

title: 记一次 Cassandra 在 k8s 环境下的数据中心迁移
date: 2022-05-14 20:20:24
tags: [Cassandra, k8s]
categories: [技术]

---



## 背景

之前使用 [cass-operator](https://github.com/k8ssandra/cass-operator) 在 k8s 上搭建了一个 cassandra 集群用来做存储，随着业务数据的增长，之前申请的 100 G 的 PVC 大小已经是捉襟见肘



急需更换到更大的磁盘空间上



最开始的时候其实有考虑到 pvc 扩容的可能，特地申请了一个[允许扩展](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/)容量的存储类型(storage class)

但是后来发现，如果直接修改对应 pvc, 整个集群的资源实际上还是由 cass-operator 管理的，这意味着一旦重启、修改 datacenter 配置项，可能会把容量给改回去，有不小的可能性直接翻车。



cass-operator 本身提供添加 node 来横向扩容的特性，我们可以直接添加集群中 cassandra 实例的数量， cass-operator 会帮我们把数据重新平衡到新的节点上，并在数据移动完成后自动执行 `nodetool cleanup` 任务，从而清理出磁盘空间。



不幸的是，目前 cass-operator [还没有提供](https://github.com/k8ssandra/cass-operator/issues/263)一个简单的方式来扩容 pvc



但这个方法只是缓兵之计，因为这会造成 cpu/内存 资源上的浪费，这两个资源并没有到达瓶颈。当我们调整完磁盘的容量后，其实是可以通过这个方式来扩容的。





当然我们也可以通过手动导出 sstable 的方式，在同一拓扑的其他集群上装载数据，但是这个方法有需要先停机维护，可能需要几个小时的时间，会影响生产环境的正常使用



所以，经过广泛的查阅文档 / google / stackoverflow ，对比后发现还是使用迁移数据中心的方式比较轻松

- 创建新 dc 集群
- 修改keyspcae配置参数, 迁移数据
- 修改客户端连接参数
- 验证数据，清理旧集群





## 开始迁移

首先准备新 dc 文件，和旧 dc 文件基本相同，修改的部分只有 `metadata` 中的 datacenter 名称，以及 pvc 申请的容量

部分和业务相关的信息用中括号标明，实际使用时请按自己的情况做替换

这里旧集群是 dc1, 新集群是 dc2

```yaml
apiVersion: cassandra.datastax.com/v1beta1
kind: CassandraDatacenter
metadata:
  name: dc2
  namespace: cassandra
spec:
  podTemplateSpec:
    spec:
      containers:
        - name: "cassandra"
          image: "<privte-domain>/k8ssandra/cass-management-api:4.0.1"
          terminationMessagePath: "/dev/other-termination-log"
          terminationMessagePolicy: "File"
  clusterName: <my-cluster-name>
  serverType: cassandra
  serverVersion: "4.0.1"
  managementApiAuth:
    insecure: {}
  size: 3
  resources:
    requests:
      memory: 24Gi
      cpu: 3
    limits:
      memory: 24Gi
      cpu: 3
  storageConfig:
    cassandraDataVolumeClaimSpec:
      storageClassName: gp2-wffc
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1T
  config:
    cassandra-yaml:
      authenticator: org.apache.cassandra.auth.PasswordAuthenticator
      authorizer: org.apache.cassandra.auth.CassandraAuthorizer
      role_manager: org.apache.cassandra.auth.CassandraRoleManager
    jvm-server-options:
      initial_heap_size: 12G
      max_heap_size: 22G
```

将上述文件 `kubectl applly -f` 应用到集群中，等待 cassdanra 新的实例都启动后，我们就可以尝试连接新 dc 的实例了。

用户名和密码都是使用之前 secrect 里面的同一套

直接连接的话，大概率会报错

```
Cannot achieve consistency level LOCAL_ONE info={required_replicas 1 alive_replicas: 0, consistency: QUORUM }
```

参见对应文档：https://docs.datastax.com/en/security/5.1/security/secSystemKeyspace.html

我们需要修改系统认证用到的 keyspace 的副本分布

```sql
ALTER KEYSPACE system_auth
    WITH REPLICATION= {'class' : 'NetworkTopologyStrategy', 
                       'dc1' : 3, 
                       'dc2' : 3};
```

修改完成后，需要手动运行 repair 任务

```
nodetool repair --full system_auth
```

这样我们就能正常登录到新 dc 的实例了。



## 迁移数据



将自己定义的 keyspace 分配副本的策略进行修改，我是在 akka projection 场景下使用，所以会涉及到这么几个 keyspcae 需要修改

原来的分布是在 dc1 集群上存储有3个副本，现在需要在 dc2 上面也存储 3 个副本

```sql
ALTER KEYSPACE akka
    WITH REPLICATION= {'class' : 'NetworkTopologyStrategy', 
                       'dc1' : 3, 
                       'dc2' : 3};

ALTER KEYSPACE akka_projection
    WITH REPLICATION= {'class' : 'NetworkTopologyStrategy', 
                       'dc1' : 3, 
                       'dc2' : 3};

ALTER KEYSPACE akka_snapshot
    WITH REPLICATION= {'class' : 'NetworkTopologyStrategy', 
                       'dc1' : 3, 
                       'dc2' : 3};                     
```

同样地，我们在修改完 keyspace 之后也需要做修复工作

```
nodetool repair --full -tr akka
nodetool repair --full -tr akka_projection
nodetool repair --full -tr akka_snapshot
```

当然也可以直接 `nodetool repair --full -tr` 修复所有 keyspace, 但是在实际过程中，修复的时候老是有 k8s node 重启，引发修复失败，所以我拆成较小的部分修复。



修复过程中可以通过 `nodetool status` 观察各个节点加载的数据大小，用来检验数据是否被正常加载到新集群。

数据修复的时间，和数据大小已经网速有关，我这里迁移 240 G 数据大约花费了一个小时



## 客户端配置修改



通过 java-driver 连接需要修改连接的数据中心

对于 datastaax-java-driver 需要修改

 `datastax-java-driver.basic.load-balancing-policy.local-datacenter`

对于 quill 需要修改

 `session.basic.load-balancing-policy.local-datacenter`

这两个配置项需要指定到新的 dc2 ，配置修改后重启 deployment 以生效



重启完成后，保险起见可以再次运行一遍 `nodetool repair` 以确保数据完整



此时可以观察到 dc1 的网络流量，cpu使用情况应该都比较低了，因为所有的请求都是从 dc2 走



## 验证和清理

我们首先把修改业务表副本拓扑分配

```
ALTER KEYSPACE akka
    WITH REPLICATION= {'class' : 'NetworkTopologyStrategy', 
                       'dc2' : 3};

ALTER KEYSPACE akka_projection
    WITH REPLICATION= {'class' : 'NetworkTopologyStrategy', 
                       'dc2' : 3};

ALTER KEYSPACE akka_snapshot
    WITH REPLICATION= {'class' : 'NetworkTopologyStrategy', 
                       'dc2' : 3};
```

这次修改就不用运行修复任务了，因为之后我们会把整个旧集群都直接删除。



临时停用旧集群节点以验证服务可用性，这里我没有想到好办法能不删除 pvc 停用，用的是手动替换镜像的方式，比如把 `/cass-management-api:4.0.1 ` 替换成一个可用的 `aws-cli:2.4.17` 镜像，这样 pod 能启动，但是对应的健康检查过不去，外部请求到旧集群实际不可用



临时停用节点

```
722740969534.dkr.ecr.cn-northwest-1.amazonaws.com.cn/k8ssandra/cass-management-api:4.0.1 

722740969534.dkr.ecr.cn-northwest-1.amazonaws.com.cn/bitnami/aws-cli:2.4.17
```

在此情况下调用接口以验证数据是否正常



验证完毕后，修改认证使用到的 keyspace ，移除 dc1 的副本

```
ALTER KEYSPACE system_auth
    WITH REPLICATION= {'class' : 'NetworkTopologyStrategy', 
                       'dc2' : 3};
```



以上验证完毕后就可以回收资源了

使用 `kubectl delete -f`删除对应的 dc 

使用 nodetool 删除节点

```
cassandra@dc2-default-sts-0:/$ nodetool status
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load        Tokens  Owns (effective)  Host ID                               Rack   
DN  10.0.45.205  46.74 GiB   1       0.0%              821d2534-0ea9-4c7c-a312-0e6047569503  default
DN  10.0.59.211  65.04 GiB   1       0.0%              acceeaa7-2342-430b-82e1-32d08a558fb4  default
DN  10.0.56.165  67.34 GiB   1       0.0%              966fb0fb-2079-41cf-a0d9-9329bc3a53a6  default
Datacenter: dc2
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load        Tokens  Owns (effective)  Host ID                               Rack   
UN  10.0.57.90   139.74 GiB  1       100.0%            da7a2578-6089-4065-a1e7-8db91e2dc045  default
UN  10.0.58.35   147.21 GiB  1       100.0%            cccfbcac-35eb-4d2a-aff4-263b98bb7c1d  default
UN  10.0.62.14   148.9 GiB   1       100.0%            b2059b41-7899-430a-a386-fa43f3805a8e  default
```

```bash
nodetool removenode 821d2534-0ea9-4c7c-a312-0e6047569503
nodetool removenode acceeaa7-2342-430b-82e1-32d08a558fb4
nodetool removenode 966fb0fb-2079-41cf-a0d9-9329bc3a53a6
```



至此，所有迁移工作完成撒花



## 参考链接

1. [cass-operator 文档: 零停机迁移 cassandra 数据库][1]
2. [datastax 文档: 在集群中添加新的数据中心][2]
3. [medium: 在 k8s 集群间迁移 cassandra][3]







[1]: https://k8ssandra.io/blog/tutorials/cassandra-database-migration-to-kubernetes-zero-downtime/

[2]: https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/operations/opsAddDCToCluster.html
[3]: https://medium.com/flant-com/migrating-cassandra-between-kubernetes-clusters-ae4ab4ada028






