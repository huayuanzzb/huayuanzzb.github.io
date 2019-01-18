---
layout: post
title:  PostgreSQL
date:   2019-01-16 15:49:26 +0800
categories: postgresql
nav_order: 3
has_children: true
permalink: /docs/ui-components
---

### 简介
OIDs是Object identifiers的缩写，用于PG内部表示系统表主键。除非用户创建表时使用with oids语句或者default_with_oids参数设置为on，否则用户创建的表不会启用oid。

### 使用
为方便使用，oid定义了一些别名：

Name | References | Description | Value Example
---|---|---|---
oid | any | numeric object identifier | 564182
regproc	| pg_proc |	function name |	sum
regprocedure |	pg_proc	| function with argument types |	sum(int4)
regoper	| pg_operator |	operator name |	+
regoperator	| pg_operator |	operator with argument types |	*(integer,integer) or -(NONE,integer)
regclass	| pg_class |	relation name |	pg_type
regtype	| pg_type |	data type name |	integer

oid允许带schema的查询。

### 例子
```
postgres=# select 'test'::regclass::int;
 int4
-------
 16394
(1 row)

postgres=# select 'public.test'::regclass::int;
 int4
-------
 16394
(1 row)

postgres=# select 16394::regclass;
 regclass
----------
 test
(1 row)
```
### 引用
https://www.postgresql.org/docs/8.1/datatype-oid.html
