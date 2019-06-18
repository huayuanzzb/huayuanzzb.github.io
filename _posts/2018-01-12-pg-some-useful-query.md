---
layout: post
title:  "一些有用的查询"
author: recaton
categories: [ PostgreSQL ]
#image: "https://images.unsplash.com/photo-1541544537156-7627a7a4aa1c?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=a20c472bc23308e390c8ffae3dd90c60&auto=format&fit=crop&w=750&q=80"
---

> 一些有用的查询语句。

查询表占用空间
```sql
select
    c.relname,
    pg_size_pretty(pg_table_size(c.relname::text))
from pg_class c
LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
where c.relkind IN ('r')
AND n.nspname <> 'pg_catalog'
AND n.nspname <> 'information_schema'
AND n.nspname !~ '^pg_toast'
order by pg_table_size(c.relname::text) desc;
```

查询表在磁盘上的存放位置
```sql
-- postgresql 使用oid作为磁盘上文件名
SELECT pg_relation_filepath('tablename');
```
查找所有没有主键的表
```sql
SELECT table_name
FROM information_schema.tables
WHERE (table_catalog, table_schema, table_name) NOT IN
          (SELECT table_catalog, table_schema, table_name
               FROM information_schema.table_constraints
               WHERE constraint_type = 'PRIMARY KEY')
AND table_schema NOT IN ('information_schema', 'pg_catalog');
```
从现有表复制一个空表出来
```sql
-- SQL标准方法, 该法不能复制出主键、索引等定义
create table new_table as select * from old_table where 1 = 2;
-- Postgresql扩展的方法，可以保留原表的列定义，约束等
create table new_table (like old_table including constraints including defaults including indexes);
```

临时禁用index
```sql
select indexrelid
from pg_stat_all_indexes
where indexrelname = 'order_odrer_number_owner_id_state_idx';

update pg_index set indisvalid='f' where indexrelid = '923778';
```

死锁相关
```sql
-- 查询
select *
from pg_stat_activity aa,
    (select a.locktype,a.database,a.pid,a.mode,a.relation,b.relname
     from pg_locks a join pg_class b on a.relation=b.oid) bb
where aa.pid=bb.pid and aa.waiting='t';
-- 取消某个查询
select pg_cancel_backend(123);
-- 不能取消时，强制剔除某个查询
select pg_terminate_backend(123);
```
去掉一个用户的连接数限制
```sql
ALTER ROLE xxxxxx CONNECTION LIMIT -1;
```
生成随机字符串
```sql
-- 生成8位随机字符
select string_agg(substr('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789', ceil(random() * 62)::integer, 1), '') FROM generate_series(1, 8);
```
