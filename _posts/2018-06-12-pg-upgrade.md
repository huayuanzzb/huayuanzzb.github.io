---
layout: post
title:  "PostgreSQL 大版本升级"
author: recaton
categories: [ PostgreSQL ]
---
> PostgreSQL 小版本升级较为简单，本文着重介绍 pglogical。

PostgreSQL 提供了多种多样的升级方法，用户可以根据不同的场景选择适合的方法。

## 简单对比
各种升级方法的简单对比：

升级方法 | 适用场景 
---|---
pg_dumpall | 数据量小，允许短暂停机 
pg_upgrade  | 有一定数据量，允许短暂停机 
pglogical  | 有一定数据量，停机时间要求很短 

## pglogical
[pglogical](https://www.2ndquadrant.com/en/resources/pglogical/pglogical-docs/) 是 PostgreSQL 的一个逻辑复制扩展，基于发布/订阅模型。

##### 限制条件
1. provider/subscriber 版本至少是 PostgreSQL 9.4；
2. 当前使用和管理 pglogical 需要 ```superuser``` 权限，在以后的版本中可能更加细化；
3. 像物理流复制一样，```UNLOGGED TABLE``` 和 ```TEMPORARY TABLE```不能复制；
4. provider/subscriber 是基于 ```database``` 工作的，如果要设置多个 ```database```，则需要为每个 ```database``` 分别设置 provider/subscriber；
<!-- todo 待验证 -->
5. 被复制的表必须有主键，否则只能复制该表的 ```insert``` 操作；
6. provider 和 subscriber 的编码必须一致，推荐使用 UTF-8；
7. provider 和 subscriber 上的表结构必须一致，subscriber 上的 constraints 必须跟 provider 上相同或者更宽松。

##### 用法
provider：PostgreSQL 9.5，ip：10.30.128.79  
subscriber：PostgreSQL 10.6，ip：10.30.128.110

###### 安装 pglogical

```shell
# provider 上安装 pglogical
root@10.30.128.79:~# apt install postgresql-9.5-pglogical/xenial-pgdg
# subscriber 上安装 pglogical
root@10.30.128.110:~# apt install postgresql-10-pglogical/xenial-pgdg
```
###### 修改配置
* provider

```shell
# 修改 postgresql.conf 中关键参数，并重启。
root@10.30.128.79:~# cat /etc/postgresql/9.5/main/postgresql.conf
wal_level = logical
shared_preload_libraries = 'pglogical'
# 如果 subscriber 从多个 provider 获取数据，可打开该参数用于处理冲突，仅支持 PostgreSQL 9.5 及以上版本
track_commit_timestamp = on
``` 
* subscriber

```shell
# 修改 postgresql.conf 中关键参数，并重启。
root@10.30.128.79:~# cat /etc/postgresql/10/main/postgresql.conf
shared_preload_libraries = 'pglogical'
```
###### 复制 provider 表结构到 subscriber

```shell
root@10.30.128.79:~# sudo -iu postgres pg_dumpall --schema-only -f dump.sql
root@10.30.128.110:~# sudo -iu postgres psql -f dump.sql
```
###### 创建 extension

```shell
# provider 和 subscriber 都执行
test=# create extension pglogical;
CREATE EXTENSION
```
###### 配置 superuser 无密码访问

* provider
```shell
# 允许 subscriber 以 superuser，replication 连接 provider
root@10.30.128.79:~# cat /etc/postgresql/9.5/main/pg_hba.conf
host    all             postgres        10.30.128.110/32               trust
host    replication     postgres       10.30.128.110/32                 trust
```
* subscriber
```shell
# 允许 subscriber 以 superuser 连接
root@10.30.128.110:~# cat /etc/postgresql/10/main/pg_hba.conf
host    all             postgres             127.0.0.1/32            trust
```

###### 开启replication
```shell
# provider
test=# SELECT pglogical.create_node(node_name := 'provider', dsn := 'host=10.30.128.79 port=5432 dbname=test');
 create_node
-------------
  3171898924
(1 row)
test=# SELECT pglogical.replication_set_add_all_tables('default', ARRAY['public']);
 replication_set_add_all_tables
--------------------------------
 t
(1 row)
# subscriber
test=# SELECT pglogical.create_node(node_name := 'subscriber', dsn := 'host=localhost port=5432 dbname=test');
 create_node
-------------
  2941155235
(1 row)
test=# SELECT pglogical.create_subscription( subscription_name := 'subscriber', provider_dsn := 'host=10.30.128.79 port=5432 dbname=test');
 create_subscription
---------------------
          2941155235
(1 row)
```
此时 subscriber 端的 log 中有类似以下输出。

```shell
2019-06-18 02:17:39.048 UTC,"postgres","test",25049,"127.0.0.1:55298",5d084865.61d9,23,"COPY",2019-06-18 02:11:49 UTC,7/11156,29829,LOG,00000,"duration: 4693.248 ms  statement: COPY......
``` 
如果是大表，```COPY``` 可能需要较长时间。

```COPY``` 完成之后，pglogical 会监听 replication set 中所有表的 dml 操作，并同步至 subscriber 中。

###### 升级
subscriber 数据同步搭建完成后，升级就很简单。在搭建 subscriber 数据同步时，没有同步 sequence， 原因如下：
1. [pglogical 文档](https://www.2ndquadrant.com/en/resources/pglogical/pglogical-docs/) 中 ```4.10 Sequences``` 部分指出：sequence 的同步不是实时的，而是周期性的。所以 subscriber 上 sequence 可能落后很多。有一些资料显示这个周期为60 ~ 70s，测试发现并不是，sequence很久都不能同步成功，不知是否是bug。
2. 虽然可以使用```select pglogical.synchronize_sequence('public.seq_test');```手动同步，但该 function 一次只能同步一个 sequence，sequence 较多时相对麻烦。

```shell
# 第三个参数为 true，立即同步所有 sequence
test=# SELECT pglogical.replication_set_add_all_sequences('default', ARRAY['public'], true);
 replication_set_add_all_tables
--------------------------------
 t
(1 row)
# subscriber 端断开订阅并删除 node
test=# select * from pglogical.drop_subscription('subscriber');
test=# select * from pglogical.drop_node('subscriber');
# provider 端删除 node
test=# select * from pglogical.drop_node('provider');
```
##### 遗留问题
sequence 同步还有一个[值自动增加1000的问题](https://github.com/2ndQuadrant/pglogical/issues/163)，某些对 sequence 值敏感的应用可能会受较大影响。对于这些应用，需要自行创建并设置 sequence 初始值。

