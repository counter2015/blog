# Django 从SQLite迁移到PostgreSQL

title: Django 从SQLite迁移到PostgreSQL
date: 2020-01-15 21:10:50
tags: [Python, Django, SQLite, PostgreSQL] 
categories: [技术, 部署]

------

## 问题背景

创建Django项目时，默认创建的数据库是SQLite，这可以方便我们进行原型开发。
但是，随着项目的进行和需求的变化，就逐渐变得不再适合了。

我遇到的问题就是
- SQLite不能支持[超过999的字段](https://stackoverflow.com/questions/24510707/is-there-any-limit-on-sqlite-query-size)
- SQLite大量数据写入时表现很差

稍微查了下[资料](https://www.digitalocean.com/community/tutorials/sqlite-vs-mysql-vs-postgresql-a-comparison-of-relational-database-management-systems)对比了下目前我知道的两个数据库


周围有人不听地吹PostgreSQL（以下简称pg）的特性多么多么强大，那就用起来吧。
其实之前也有尝试着[搭过](https://counter2015.com/2019/08/22/postgresql/),算是一回生，二回熟了？


以下步骤基于如下环境
-  系统 CentOS 7
-  数据库 PostgreSQL 12
-  Django 2.2
-  Python 3.6



> 迁移前切记，先备份数据库。



## PostgreSQL下载安装

之前在ubuntu环境下安装过pg,现在简单讲下在CentOS环境下安装步骤

参考： https://www.postgresql.org/download/linux/redhat/ 

```shell
$ yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
$ yum install postgresql12
$ yum install postgresql12-server
$ /usr/pgsql-12/bin/postgresql-12-setup initdb
$ systemctl enable postgresql-12
$ systemctl start postgresql-12
```



设置对应的表和用户/密码/权限

```shell
$ su postgres
$ cd 
$ psql
postgres=# create database mydb;
CREATE DATABASE
postgres=# create user myuser with encrypted password 'mypass';
CREATE ROLE
postgres=# grant all privileges on database mydb to myuser;
GRANT
```



修改配置，允许外部访问

```shell
# 需要修改配置文件 postgresql.conf, 确定其位置有两个方法

# 方法一
$ updatedb
$ locate postgresql.conf
/usr/pgsql-12/share/postgresql.conf.sample
/var/lib/pgsql/12/data/postgresql.conf

# 方法二
$ find / -name "postgresql.conf"
/var/lib/pgsql/12/data/postgresql.conf
```

找到如下部分的配置文件

```shell
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

#listen_addresses = 'localhost'         # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
#port = 5432                            # (change requires restart)
max_connections = 100                   # (change requires restart)
```

修改`listen_addresses`配置项（同时去除注释）为

` listen_addresses = '*' `
修改`port`配置项（同时去除注释）为 15432

这里是我为了防止可能的端口冲突，可以根据需要自己选择是否修改

修改配置文件

```shell
$ vim /var/lib/pgsql/12/data/pg_hba.conf

# user define config
host    all             all             0.0.0.0/0               md5

```

在文件末尾添加如下一行

这将会允许从外部所有地址的连接(注意潜在的安全问题)。


修改完成后，需要重启数据库以使配置文件生效
```shell
$ systemctl restart postgresql-12 
```

这样就可以从客户端远程访问数据库了（比如说服务端的host为mycentos）
```shell
$ psql -h mycentos -p 15432 -U myuser -d mydb
Password for user myuser: 
psql (11.5 (Ubuntu 11.5-3.pgdg18.04+1), server 12.1)
WARNING: psql major version 11, server major version 12.
         Some psql features might not work.
Type "help" for help.

mydb=> 
```





## 可选： 数据可视化工具的安装

这方面最有名的工具就是[Navicat](https://www.navicat.com/en/)了，但是我不用,因为（[没钱]( https://www.navicat.com/en/store/navicat-premium-plan )

下面介绍的是开源的[dbeaver](https://dbeaver.io/)（海狸）
![](https://counter2015.com/picture/dbeaver-logo-1.png)

用起来感觉还不错
![](https://counter2015.com/picture/dbeaver-1.jpg)

## 数据和表的迁移

分为以下几个步骤

1. 将数据从SQLite中导出成文件

2. 在PostgreSQL中重新创建对应的表结构

3. 将数据文件导入到PostgreSQL中

4. 将Django的后端数据库驱动切换成PostgreSQL,并配置对应的数据库连接信息

导出可以借助Django自带的工具

```shell
$ python3 manage.py dumpdata > datadump.json
```

安装Python - PostgreSQL驱动

```shell
$ pip3 install psycopg2-binary
```

修改`settings.py`

```shell
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypass',
        'HOST': 'mycentos',
        'PORT': '15432'
    }
}
```

执行表结构的迁移

```shell
# 你可能需要激活虚拟环境
$ source env/bin/activate
$ python3 manage.py makemigrations
$ python3 manage.py migrate
```

数据的导入

```shell
$ python3 manage.py loaddata datadump.json
```

这里我遇到了两个小坑

```shell
django.db.utils.IntegrityError: Problem installing fixtures: 错误:  插入或更新表 "predicts" 违反外键约束 "predicts_dataset_id_9ffd0177_fk_mark_datasets_id"
DETAIL:  键值对(dataset_id)=(3)没有在表"datasets"中出现.
```

这是SQLite对级联删除的支持问题

尽管我在`models.py`中声明了
`models.ForeignKey(DataSets, on_delete=models.CASCADE, null=True, blank=True)`

但是从数据库直接删除（而不是通过Django 的ORM （ Object Relational Mapping ）删除）的时候，并没有做级联删除
<del>于是我还得手动删除</del>

```shell
psycopg2.errors.UniqueViolation: 错误:  重复键违反唯一约束"django_content_type_app_label_model_76bd3d3b_uniq"
DETAIL:  键值"(app_label, model)=(auth, permission)" 已经存在
```

这里有两种解决方法

方法一：重新导出数据

```shell
$ python3 manage.py dumpdata --natural-primary --natural-foreign > data.json
$ python3 manage.py loaddata data.json
```



方法二：删除`content_type`（没搞懂原理）

```shell
$ python3 manage.py shell
from django.contrib.contenttypes.models import ContentType
ContentType.objects.all().delete()
```
使用上面任意一个方法后，重新加载数据
```shell
$ python3 manage.py loaddata datadump.json
```



如果有使用raw SQL操作数据库，需要对比两个数据库的不同API，进行业务代码层面的修改

这里我用的都是ORM操作，所以没有这个问题。



至此，迁移工作就基本完成了。



## 参考链接

1. [How to migrate your Django project from SQLite to PostgreSQL][1]
2. [Creating user, database and adding access on PostgreSQL][2]
3. [How to enable remote access to PostgreSQL server on a Plesk server?][3]
4. [HowTo Safely Open a PostgreSQL Port for Remote Access?][4]
5. [PostgreSQL doc: The pg_hba.conf File][5]
6. [Postgres login: How to log into a Postgresql database][6]
7. [How To Use PostgreSQL with your Django Application on Ubuntu 14.04][7]
8. [Django when I try to migrate migrations to PostgreSQL it throws a exceptions psycopg2.ProgrammingError][8]



[1]: https://www.vphventures.com/how-to-migrate-your-django-project-from-sqlite-to-postgresql/
[2]: https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e
[3]: https://support.plesk.com/hc/en-us/articles/115003321434-How-to-enable-remote-access-to-PostgreSQL-server-on-a-Plesk-server-
[4]: http://www.project-open.com/en/howto-postgresql-port-secure-remote-access
[5]: https://www.postgresql.org/docs/12/auth-pg-hba-conf.html
[6]: https://alvinalexander.com/blog/post/postgresql/log-in-postgresql-database
[7]: https://www.digitalocean.com/community/tutorials/how-to-use-postgresql-with-your-django-application-on-ubuntu-14-04
[8]: https://stackoverflow.com/questions/45872470/django-when-i-try-to-migrate-migrations-to-postgresql-it-throws-a-exceptions-psy





  

  

  