# PostgreSQL 快速使用指南

title: PostgreSQL 快速使用指南
date: 2019-08-22 12:01:30
tags: [PostgreSQL, SQL] 
categories: [技术, 部署]

------



## What and Why

PostgreSQL是一个功能强大的开源对象关系数据库系统，它使用和扩展了SQL语言，并结合了许多安全存储和扩展最复杂数据工作负载的功能。PostgreSQL的起源可以追溯到1986年，作为加州大学伯克利分校POSTGRES项目的一部分，并在核心平台上进行了30多年的积极开发。

PostgreSQL提供了许多功能，旨在帮助开发人员构建应用程序，管理员保护数据完整性并构建容错环境，并帮助您管理数据，无论数据集有多大或多小。除了免费和开源之外，PostgreSQL还具有高度可扩展性。例如，您可以定义自己的数据类型，构建自定义函数，甚至可以编写不同编程语言的代码，而无需重新编译数据库！


## 环境和版本

不同的环境和版本，安装过程也略有不同，具体可以查看[官网](https://www.postgresql.org/download/)这里选用的是 

- PostgreSQL 11
- OS: Ubuntu 18.04 bionic [Ubuntu on Windows 10]
- Kernel: x86_64 Linux 4.4.0-17763-Microsoft
- 以下命令**均为root用户**操作



## 安装

在`/etc/apt/sources.list.d/pgdg.list`添加如下pgsql远程仓库配置

```text
deb http://apt.postgresql.org/pub/repos/apt/ bionic-pgdg main
```



导入仓库校验密钥，更新apt包列表

```plain
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

$ apt-get update
```



安装pgsql11

```
$ apt-get install postgresql-11 pgadmin4
$ apt install postgresql-common
$ sh /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```



## 基本使用

```plain
# 启动postgresql
$ service postgresql start

# 切换至postgres用户（安装成功时会自动创建该用户）
$ sudo su - postgres

# 进入psql命令行, 查看当前角色
$ psql
postgres=# SELECT rolname FROM pg_roles;

          rolname          
---------------------------
 postgres
 pg_monitor
 pg_read_all_settings
 pg_read_all_stats
 pg_stat_scan_tables
 pg_read_server_files
 pg_write_server_files
 pg_execute_server_program
 pg_signal_backend
(9 rows)
```

可以看到，在角色列表中，没有root用户，如果我们希望能让root用户也能访问创建数据库表的话，就需要先添加上。



我们可以给不同的角色分配不同的表，函数，读写权限，这里为了方便起见，创建的root用户具有管理员权限,同时添加了登录权限。

```pgsql
postgres=# CREATE ROLE root SUPERUSER;
CREATE ROLE

postgres=# ALTER ROLE root LOGIN;
ALTER ROLE
```

如此一来，root用户就能正常创建数据库了，

```text
$ createdb mydb
WARNING:  could not flush dirty data: Function not implemented
```

上面这个命令会报warning是由于WSL和pgsql的兼容问题，暂未发现会影响使用的地方。



接下来，进入刚创建的数据库，尝试下面的命令。

```pgsql
$ pgsql mydb
psql (11.5 (Ubuntu 11.5-1.pgdg18.04+1))
Type "help" for help.

mydb=# SELECT version();

                                                              version                                                              
------------------------------------------------------------------------------------
 PostgreSQL 11.5 (Ubuntu 11.5-1.pgdg18.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 7.4.0-1ubuntu1~18.04.1) 7.4.0, 64-bit
(1 row)


mydb=# SELECT current_date;
 current_date 
--------------
 2019-08-20
(1 row)

mydb=# SELECT 123456789 * 987654321;
ERROR:  integer out of range
mydb=# SELECT 123 * 987;
 ?column? 
----------
   121401
(1 row)

mydb=# SELECT 4 / 3;
 ?column? 
----------
        1
(1 row)

mydb=# SELECT 4 / 0;
ERROR:  division by zero


```



使用转义符`\`可以运行内部命令 `\h`可以查看帮助 `\q`可以退出，其他就不详述了，详细可参考[官方文档](https://www.postgresql.org/docs/current/)

```pgsql
mydb=# \h
mydb=# \h ALTER


mydb=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 mydb      | root     | UTF8     | C.UTF-8 | C.UTF-8 | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)

mydb=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 root      | Superuser                                                  | {}
 
 
 mydb-# \dt
        List of relations
 Schema |  Name   | Type  | Owner 
--------+---------+-------+-------
 public | cities  | table | root
 public | weather | table | root
(2 rows)

```



### 建表语句

```pgsql
CREATE TABLE weather (
    city            varchar(80),
    temp_low         int,            -- low temperature
    temp_high         int,           -- high temperature
    prcp            real,            -- precipitation
    date            date
);

CREATE TABLE cities (
    name            varchar(80),
    location        point
);
```

psql可以任意使用空白字符（空格、制表符、换行）。

`--`后面该行的语句为注释，会被忽略。

`point`为psql的数据类型，是一个二元实数组，如`(2.1,3.3)`

### 删表语句

```pgsql
DROP TABLE tablename;
```

{% spoiler 删库跑路现在已经不流行了，现在流行开源跑路 %}

### 插入数据

```pgsql
# 按指定顺序插入
INSERT INTO weather VALUES ('San Francisco', 46, 50, 0.25, '1994-11-27');

INSERT INTO cities VALUES ('San Francisco', '(-194.0, 53.0)');

# 指定名称插入
INSERT INTO weather (city, temp_lo, temp_hi, prcp, date)
    VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29');
    
# 修改字段顺序，并只选择部分字段插入
INSERT INTO weather (date, city, temp_hi, temp_lo)
    VALUES ('1994-11-29', 'Hayward', 54, 37);
    
COPY weather FROM '/tmp/weather.txt' DELIMITER ',' CSV;
```

其中`/tmp/weather.txt`的内容如下

```shell
a, 45, 50, 0.25, 1996-12-22
```





查询语句(这里后来我又往表里加了几条数据，和上面的有出入),参见[官方文档](https://www.postgresql.org/docs/current/tutorial-select.html)

```pgsql
mydb=# SELECT * FROM weather;
     city      | temp_low | temp_high | prcp |    date    
---------------+----------+-----------+------+------------
 San Francisco |       46 |        50 | 0.25 | 1994-11-27
 San Francisco |       43 |        57 |    0 | 1994-11-29
 Hayward       |       37 |        54 |      | 1994-11-29
 a             |       45 |        50 | 0.25 | 1996-12-22

mydb=# SELECT city, temp_low, temp_high, prcp, date FROM weather;
     city      | temp_low | temp_high | prcp |    date    
---------------+----------+-----------+------+------------
 San Francisco |       46 |        50 | 0.25 | 1994-11-27
 San Francisco |       43 |        57 |    0 | 1994-11-29
 Hayward       |       37 |        54 |      | 1994-11-29
 a             |       45 |        50 | 0.25 | 1996-12-22
(4 rows)


mydb=# SELECT city, (temp_high+temp_low)/2 AS temp_avg, date FROM weather;
     city      | temp_avg |    date    
---------------+----------+------------
 San Francisco |       48 | 1994-11-27
 San Francisco |       50 | 1994-11-29
 Hayward       |       45 | 1994-11-29
 a             |       47 | 1996-12-22

mydb=# SELECT * FROM weather
mydb-#     WHERE city = 'San Francisco' AND prcp > 0.0;
     city      | temp_low | temp_high | prcp |    date    
---------------+----------+-----------+------+------------
 San Francisco |       46 |        50 | 0.25 | 1994-11-27
(1 row)

mydb=# SELECT * FROM weather
mydb-#     ORDER BY city;
     city      | temp_low | temp_high | prcp |    date    
---------------+----------+-----------+------+------------
 Hayward       |       37 |        54 |      | 1994-11-29
 San Francisco |       46 |        50 | 0.25 | 1994-11-27
 San Francisco |       43 |        57 |    0 | 1994-11-29
 a             |       45 |        50 | 0.25 | 1996-12-22


mydb=# SELECT * FROM weather
    ORDER BY city DESC;
     city      | temp_low | temp_high | prcp |    date    
---------------+----------+-----------+------+------------
 a             |       45 |        50 | 0.25 | 1996-12-22
 San Francisco |       46 |        50 | 0.25 | 1994-11-27
 San Francisco |       43 |        57 |    0 | 1994-11-29
 Hayward       |       37 |        54 |      | 1994-11-29
(4 rows)

```



> 尽管select*对于临时查询很有用，但在生产代码中它被广泛认为是不好的样式，因为当表的列发生变动时，会更改查询返回的结果。
>
> 
>
> 在一些数据库系统中，包括旧版本的PostgreSQL，distinct的实现自动对行进行排序，因此ORDER BY是不必要的。但这不是SQL标准所要求的，当前的PostgreSQL不保证distinct会导致行被排序。



如此一来，我们可以<del>自豪地</del>说，“我也是个(c)数(r)据(u)库(d)工(b)程(o)师(y)啦！”

