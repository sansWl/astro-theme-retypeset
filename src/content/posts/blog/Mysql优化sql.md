---
title: Mysql优化
published: 2025-08-11
tags:
- Mysql
- SQL
lang: zh
abbrlink: mysql-optimize-sql
---
### 分析sql
MySQL慢查询日志
>1. 修改配置文件 `slow _query_log` <br>
>2. `SET GLOBAL slow_query_log = 1;` 开启慢查询日志 session 级别 <br>
>3. 修改慢查询阈值 即定义多长时间的sql为慢sql，long_query_time(default: 10 sec)<br>


`explain`命令可以分析sql的执行计划，可以帮助我们分析sql的性能瓶颈。
语法：
```sql
#1. explain select * from table_name;

#2. explain format=json select * from table_name;
```

执行explain命令：
![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250921061230255-1941489668.png)

`type：`
>system：系统表，少量数据，往往不需要进行磁盘IO <br>
>const：常量连接 <br>
>eq_ref：主键索引(primary key)或者非空唯一索引(unique not null)等值扫描 <br>
>ref：非主键非唯一索引等值扫描 <br>
>range：范围扫描 index：索引树扫描 * <br>
>ALL：全表扫描(full table scan)<br>
> `system > const > eq_ref > ref > range > index > ALL` <br>

`key_len:`
>索引长度，影响索引存储数量<br>

`extra:`
>Extra 列包含查询优化器的额外信息。常见的值有：<br>
> ● Using where：表示查询使用了 WHERE 过滤条件,需要在server层进行过滤。<br>
> ● Using index：表示查询只使用了索引，不需要回表查询数据。<br>
> ● Using filesort：表示查询需要额外的排序操作，这是一个性能瓶颈。<br>
> ● Using temporary：表示查询使用了临时表，这是一个性能瓶颈。<br>
> 优化建议：尽量避免 Using filesort 和 Using temporary，可以通过调整查询语句、增加索引或优化表结构来消除这些性能瓶颈。<br>

`rows:`
>扫描的行数，影响查询性能,联表操作时，连接列最好保持一致<br>

`filtered:`
>表示查询条件过滤的行数占总行数的比例，可以用来判断是否需要进行优化，代表着查询条件、索引的有效性、高效性<br>

估计性能： Mysql索引选择：cost  《=》 server_cost+engine_cost <br>
`cost_info:`
> ● server_cost：优化器成本估算 服务器成本<br>
> ● engine_cost：优化器成本估算 特定存储引擎成本<br>
主查询成本详情 (cost_info)  `使用 explain format=json query 查看`  <br>
  ○ read_cost: "545.00" 数据读取成本（I/O相关）<br>
  ○ eval_cost: "17889.20" 数据计算处理成本<br>
  ○ prefix_cost: "18434.20" 当前操作及之前所有操作的累计成本<br>
  ○ data_read_per_join: "151M" 连接操作读取的数据量<br>

### 优化目标
> 根据上面 explain 的结果，我们可以得出以下结论：<br>
> 1. 保证索引的高效使用，避免索引失效，减少回表操作等
> 2. 查询操作尽可能减少数据库的额外操作，譬如避免使用 filesort 和 Using temporary <br>
> 3. 控制查询数据量级，减少过滤查询的行数 <br>

### 优化方案
 [Mysql优化 -官方文档](https://dev.mysql.com/doc/refman/8.4/en/select-optimization.html)

 优化实例：
 1. 减少IO：
````sql
-- 利用case when 分情况处理数据，减少IO消耗
update table_name 
set 
column_name1 =  case when column_name1 = 'value1' then 'value2',
column_name2 =  case when column_name2 = 'value2' then 'value3'
-- 注意 case when 如果没设置 else 则默认会将数据重设为 null，
-- 如需保持原样请使用else column_name 或者 where限制条件 

--索引覆盖，避免回表查询
-- index(name,phone)
EXPLAIN SELECT name,phone FROM user WHERE name= '陈艮' AND phone = '13601994087';
EXPLAIN SELECT name,phone FROM user WHERE name= '陈艮' AND phone = '13601994087';
EXPLAIN SELECT name FROM user WHERE name= '陈艮' AND phone = '13601994087';
EXPLAIN SELECT phone FROM user WHERE name= '陈艮' AND phone = '13601994087';
````
2. 控制查询数据量级：
```sql
-- 例如深分页,通过跳过已查询的id来减少查询数据量，逻辑和ES的 search_after 类似
-- 前提需要保证查询操作的id有序且连续，且分页期间数据不能更改以免数据不一致，ES 利用 ptr机制处理，可参考
select * from table_name limit 1000,10 where id > 1000000000;
```

3. 索引优化：
> 索引页默认大小16KB（innodb_page_size）
> 存储索引数=》16kb*1024(字节) / (point_ptr[ 6字节]+index[bigint 8字节]) ==》1170
```sql
-- 1. 索引字段选择：
-- 索引字段选择应尽量选择区分度高的字段，区分度高的字段可以有效减少索引的大小，提高查询效率。
-- 2. 索引字段类型选择：
-- 索引字段类型选择应尽量选择较小的数据类型，减少索引的大小，提高查询效率。
-- 3. 索引字段长度选择：
-- 索引字段长度选择应尽量选择较短的长度，减少索引的大小，提高查询效率。

select col1, col2, col3 from table_name where col2 = 'value2';
```
参考链接：
[MySQL索引从基础到原理](https://developer.aliyun.com/article/841106)
[explain extra 分析](https://www.cnblogs.com/myseries/p/11262054.html)