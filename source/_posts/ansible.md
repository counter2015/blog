# Ansible + AWX 安装

title: Ansible + AWX 安装
date: 2020-06-12 22:59:30
tags: [RedHat, Ansible, AWX] 
categories: [技术]

------



## Why Ansible

Ansible是一个开源的Dev-ops工具，现在已经被Red Hat 收购。

包括但不限于以下用途

- 配置管理
- 应用程序部署
- 零停机滚动更新
- 多节点编排
- ad-hoc任务执行

不幸的是，Red Hat将Ansible从EPEL中移到了他们的一个存储库中，这需要您拥有Red Hat订阅<del>交钱</del>才能访问。



## 准备工作

首先需要在控制机上安装Ansible,然后通过SSH(默认情况下)与作业机（执行自动化操作的终端设备）进行通信

如果是本地CentOS安装，有这么几个可选方案

- 从[Github仓库](https://github.com/ansible/ansible.git)编译安装
- 通过python-pip安装
- 订阅红帽的额外仓库
- 添加第三方存储库

这里为了简单起见用的最后一种方法

这里以CentOS为例，其他环境的安装步骤可以参考[官方文档](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

首先添加repo地址，在控制机上下载安装Ansible

```shell
$ cat <<EOF | sudo tee /etc/yum.repos.d/ansible.repo
[Ansible]
name = ansible
baseurl = https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/
enabled = 1
gpgcheck = 0
EOF

$ yum install ansible
$ ansible --version
ansible 2.9.9
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/user/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Apr  2 2020, 13:16:51) [GCC 4.8.5 20150623 (Red Hat 4.8.5-39)]
```

安装完成后，配置文件的位置为`/etc/ansible`

为了更加方便的进行管理，我们还需要安装[AWX](https://github.com/ansible/awx)

> Originally, Ansible Tower was called "AWX" — see [this old blog post](https://www.ansible.com/blog/2013/08/05/supercharge-ansible-with-ansibleworks-awx-1-2) from 2013. And apparently that was kind of a short-hand for 'AnsibleWorks', [the original name of the company](https://www.ansible.com/blog/2013/03/04/introducing-ansibleworks) that became Ansible, that became Ansible by Red Hat. Straight from the horse's mouth

AWX是一个开源的Web应用程序，它为Ansible提供了UI界面，REST APT和任务引擎，它是Ansible Tower的开源版本。

通过AWX，可以管理Ansible playbooks,配置列表和定时作业。

在进行安装前，软件方面首先需要满足下面的最简前置依赖条件

- Ansible 版本 2.8+

- Docker 近期版本，安装步骤可以参考[官方文档](https://docs.docker.com/engine/install/centos/)

- Docker的[Python模块](https://pypi.org/project/docker/)

  这个模块和docker-py不兼容，如果之前安装过docker-py，需要卸载

- [GNU Make](https://www.gnu.org/software/make/)

- [Git](https://git-scm.com/) 1.8.4+

- Python 3.6+

系统方面则是要满足下列条件

- 最少4GB的内存
- 最少2个CPU核心
- 最少20GB的磁盘空间
- 运行Docker,Openshift或者Kubernetes,这里选择的是Docker
- 如果选择使用PostgreSQL，最低的版本要求为10



## Ansible Quick Start

让我们在这里打断下安装AWX的步骤，先来了解下[Ansible的基本概念和功能](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html)



### 基本概念

**控制节点(Control node)**
在控制节点上安装有Ansible,需要安装Python，不支持Windows，可以有多个控制节点

**受控节点(Managed nodes)**
使用Ansible控制的终端设备，也被称为主机(hosts)，不需要安装Ansible

**清单(Inventory)**
列出受控节点的配置文件列表，比如IP地址和域名

**模块(Modules)**
Ansible执行的代码单元，每一个模块有特定的用途，你可以在Task中单独调用某个模块，也可以在Playbooks中调用多个不同的模块

**任务(Tasks)**
Ansible的动作单元

**任务列表(Playbooks)**
对于Tasks的有序列表


![Ansible简单示意图](https://counter2015.com/picture/ansible-1.png)

为了能在远程终端上执行命令，我们需要先知道IP地址，一个比较常见的做法是从`inventory`中读取配置文件，这样可以更加方便和灵活(虽然也可以简单粗暴的将IP地址传入到[ad-hoc](https://en.wikipedia.org/wiki/Ad_hoc)命令里)

首先，我们在Inventory配置文件(`/etc/ansible/hosts`)中加入Managed Node的信息，可以用ip地址或者域名来表示

```shell
10.1.1.1
10.1.1.2
```

Ansible是通过SSH和Managed Node进行通信的，默认情况下，会用本机的OpenSSH以当前用户名进行连接。

这里需要先在机器间设置SSH互信

以10.1.1.1的机器为例
```shell
ssh-keygen -t rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub 10.1.1.1
```
然后运行Ping命令
```shell
$ ansible all -m ping
10.1.1.1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
10.1.1.2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
```

假如之前没有正确设置SSH互信的话，就会返回错误信息

```shell
$ ansible all -m ping
10.1.1.1 | UNREACHABLE! => {
    "changed": false, 
    "msg": "Failed to connect to the host via ssh: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).", 
    "unreachable": true
}
10.1.1.2 | UNREACHABLE! => {
    "changed": false, 
    "msg": "Failed to connect to the host via ssh: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).", 
    "unreachable": true
}
```

经典命令hello world

```shell
$ ansible all -a "/bin/echo hello"
10.1.1.1 | CHANGED | rc=0 >>
hello
10.1.1.2 | CHANGED | rc=0 >>
hello
```

这样我们就完成了一个简单的ad-hoc的任务分发了



我们来简单解释下上面用到的参数

`ansible all -m ping`

-m MODULE_NAME, --module-name MODULE_NAME
                        module name to execute (default=command)

`ansible all -a "/bin/echo hello"`

-a MODULE_ARGS, --args MODULE_ARGS

也可以简单的指定某台机器作为Managed Node
```shell
$ ansible 10.1.2.3 -a "/bin/echo hello"
10.1.2.3 | CHANGED | rc=0 >>
hello
```

更加深入的学习可以参照[User Guide](https://docs.ansible.com/ansible/latest/user_guide)，这里就不赘述了

## 使用Docker-compose 安装 AWX

### 前提条件

- Docker已经安装完毕，服务正常运行
- [docker-compose](https://pypi.org/project/docker-compose/) Python模块
- [Docker Compose](https://docs.docker.com/compose/install/)

首先从github上把代码拉下来，切换到对应的版本分支下

```shell
$ git clone -b 11.2.0 https://github.com/ansible/awx.git
$ cd awx 
$ git checkout -b 11.2.0
```



### 安装前的准备

**部署到远程主机(可选)**

默认情况下，`installer/inventory`的配置文件会把AWX部署到本地主机，但是也可以将其部署到远程主机。

`installer/install.yml`的playbook可以用来在本地编译镜像，将构建好的镜像传输并部署到远程机器

要做到这点非常简单，只要修改`insatll/invetory`中，注释掉`localhost`并且添加远程服务器

```shell
# localhost ansible_connection=local ansible_python_interpreter="/usr/bin/env python3"
awx-server
```

这里我们就不修改了，选择把AWX部署到本地

这里可以简单看下安装使用到的playbook配置文件`installer/install.yml`

```yaml
---
- name: Build and deploy AWX
  hosts: all
  roles:
    - {role: check_vars}
    - {role: image_build, when: "dockerhub_base is not defined"}
    - {role: image_push, when: "docker_registry is defined and dockerhub_base is not defined"}
    - {role: kubernetes, when: "openshift_host is defined or kubernetes_context is defined"}
    - {role: local_docker, when: "openshift_host is not defined and kubernetes_context is not defined"}
```

这几个步骤对应的文件结构如下

```shell
$ tree -L 3
.
├── build.yml
├── install.yml
├── inventory
└── roles
    ├── check_vars
    │   └── tasks
    ├── image_build
    │   ├── defaults
    │   ├── files
    │   ├── tasks
    │   └── templates
    ├── image_push
    │   └── tasks
    ├── kubernetes
    │   ├── defaults
    │   ├── handlers
    │   ├── tasks
    │   ├── templates
    │   └── vars
    └── local_docker
        ├── defaults
        ├── tasks
        └── templates
```

**Inventory variables**

在开始前，我们需要先检查一遍`installer/inventory`中的配置项



*postgres_data_dir*

AWX的安装需要依赖PostgreSQL，默认情况下，会在容器内创建一个数据库，并且把数据文件保存在宿主机中。

这种情况下，必须将`postgres_data_dir`设置为可以安装容器的路径

如果想使用外部的数据库，那么在`installer/inventory`中，将`pg_hostname`取消注释，修改连接信息

- `pg_username`
- `pg_password`
- `pg_admin_password`
- `pg_database`
- `pg_port`



*host_port*

这个端口配置的是AWX容器映射到外部WEB服务上的端口，默认是80，如果是未定义的值，那么就不会暴露端口

这里随便设置了一个值：`host_port=10080`



*host_port_ssl*

提供一个端口号以支持SSL协议，如果是未定义的值，那么不会暴露端口，只有同时也设置了`ssl_certificate`时才有效



*ssl_certificate*

可选项，提供包含证书和私钥的文件，需要时一个`.pem`文件

以上两个设置我都用的默认值，没有修改



*docker_compose_dir*

使用docker-compose的文件夹位置



*custom_venv_dir*

从本地主机添加的自定义python venv环境，以便在安装时传递到容器中



这里为了在本地构建镜像，做了如下设置

```
docker_registry=172.30.1.1:5000
docker_registry_repository=awx
docker_registry_username=developer
```

这样构建镜像时，默认就是以`developer`用户推送到`awx`的仓库中

关于带来相关的配置

*http_proxy, https_proxy, no_proxy*保持默认值不修改



完整的配置文件如下

```shell
localhost ansible_connection=local ansible_python_interpreter="/usr/bin/env python3"

[all:vars]

dockerhub_base=ansible

awx_task_hostname=awx
awx_web_hostname=awxweb
postgres_data_dir="~/.awx/pgdocker"
host_port=10080
host_port_ssl=443
docker_compose_dir="~/.awx/awxcompose"

pg_username=awx
pg_password=awxpass
pg_database=awx
pg_port=5432

admin_user=admin
admin_password=password

create_preload_data=True

secret_key=awxsecret
```



### 进行安装

```shell
$ cd installer/
$ ansible-playbook -i inventory install.yml
```

安装可能需要较长的一段时间





## 安装后

安装后需要执行数据库的迁移，可以查看到日志

```shell
$ docker logs -f awx_task
Using /etc/ansible/ansible.cfg as config file
127.0.0.1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "elapsed": 0,
    "match_groupdict": {},
    "match_groups": [],
    "path": null,
    "port": 5432,
    "search_regex": null,
    "state": "started"
}
Using /etc/ansible/ansible.cfg as config file
127.0.0.1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "db": "awx"
}
2020-06-09 04:27:04,941 INFO     rbac_migrations Computing role roots..
2020-06-09 04:27:04,955 INFO     rbac_migrations Found 0 roots in 0.000145 seconds, rebuilding ancestry map
2020-06-09 04:27:04,955 INFO     rbac_migrations Rebuild ancestors completed in 0.000006 seconds
2020-06-09 04:27:04,955 INFO     rbac_migrations Done.
2020-06-09 04:27:10,854 INFO     rbac_migrations Computing role roots..
2020-06-09 04:27:10,855 INFO     rbac_migrations Found 0 roots in 0.000122 seconds, rebuilding ancestry map
2020-06-09 04:27:10,855 INFO     rbac_migrations Rebuild ancestors completed in 0.000021 seconds
2020-06-09 04:27:10,856 INFO     rbac_migrations Done.

```

如果安装后无法联通brdige模式的docker network，请检查是否存在冲突

可以参考此[PR](https://github.com/ansible/awx/pull/7185)进行修改,`docker_compose_subnet`配置项填入一个不会冲突的本地ip地址，如192.168.1.5/24

你可能需要先删除对应的 docker network

```shell
$ docker network rm awxcompose_default
```



迁移完成后，可以在对应端口打开awx web界面

![](https://counter2015.com/picture/ansible-2.png)

这里需要输入的密码就是之前在inventory中配置的`admin_user`和`admin_password`

主界面一览

![](https://counter2015.com/picture/ansible-3.png)





## 相关文章

[Ansible playbook 批量部署 node-exporters 服务](https://counter2015/2020/08/11/ansible-action/)



## 参考链接

1. [ansible 文档][1]
2. [Github: ansible][2]
3. [Gihub: awx][3]
4. [awx issue: How do I change the IP addresses and three subnets that get created for the docker containers][4]
5. [awx pull request: Add subnet configuration to Docker Compose to avoid conflicts.][5]
6. [RedHat: A system administrator's guide to getting started with Ansible - FAST!][6]
7. [Ansible not available in epel repo installed on RH 7][7]






[1]: https://docs.ansible.com/ansible/latest

[2]: https://github.com/ansible/ansible
[3]: https://github.com/ansible/awx
[4]: https://github.com/ansible/awx/issues/750
[5]: https://github.com/ansible/awx/pull/7185/files
[6]: https://www.redhat.com/en/blog/system-administrators-guide-getting-started-ansible-fast
[7]:  https://linuxacademy.com/community/show/21665-ansible-not-available-in-epel-repo-installed-on-rh-7/



