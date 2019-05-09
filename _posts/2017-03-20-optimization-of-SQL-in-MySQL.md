---
layout: post
title: "MySQL中SQL的调优方法"
date: 2017-03-20 23:30:00
tags: MySQL SQL 调优
---
MySQL中对SQL的调优方法

在网上查了些资料，前辈们说的都很齐全，我这针对他们所说的，根据自己的实践，整理了一些基本方法或步骤，算是对SQL调优的一个总结吧。

## 查找慢查询
查看慢查询的时间

	show variables like 'long_query_time';

临时设置慢查询的值

	set long_query_time=2

但是，如果需要永久设置，就需要在MySQL的配置文件中进行配置。

在 mysql 的配置文件中，添加

	long_query_time=2(记录sql执行超过的时间，默认为10s)
	log-slow-queries=/var/log/mysql/slowquery.log(如果为空，系统会指定一个缺省的文件 hostname_slow.log)

这样就打开了慢查询日志，mysql会将执行时间超过 2 秒的语句视作慢查询，并记录在慢查询日志中。查看日志使用工具 `mysqldumpslow`，使用 -h 选项可查看帮助信息

查看访问次数最多的20个sql

	mysqldumpslow -s c -t 20 slowquery.log

查看返回记录集最多的20个sql

	mysqldumpslow -s r -t 20 slowquery.log

按照时间返回前10条sql里含有 `left join`的语句

	mysqldumpslow -t 10 -s t -g “left join” slowquery.log
当然，如果查询慢查询日志报告，我们可以借用一些其他的工具，比如 `pt-query-digest`，这是 percona toolkit 的包，在Ubuntu下可以直接安装。

```shell
sudo apt-get install percona-toolkit
```

然后使用这个工具来查看慢查询日志，能够生成更加详细的报告。

## show status

MySQL中直接使用如下几个命令查看当前status

```shell
show status;
show full processlist;
```



## 使用 EXPLAIN

上述方法，通过慢查询日志，找到慢查询的语句之后，使用 explain 分析语句的执行过程。

explain 的使用方法很简单，就是 `explain select ... from ... [where ...]` 的方式。 输出结果为

	+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+
	| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
	+----+-------------+-------+-------+-------------------+---------+---------+-------+------+-------+

下面对各个属性进行了解：

1、id：这是SELECT的查询序列号

2、`select_type`：`select_type` 就是 select 的类型，可以有以下几种：

    SIMPLE：简单SELECT(不使用UNION或子查询等)

    PRIMARY：最外面的SELECT

    UNION：UNION中的第二个或后面的SELECT语句

    DEPENDENT UNION：UNION中的第二个或后面的SELECT语句，取决于外面的查询

    UNION RESULT：UNION的结果。

    SUBQUERY：子查询中的第一个SELECT

    DEPENDENT SUBQUERY：子查询中的第一个SELECT，取决于外面的查询

    DERIVED：导出表的SELECT(FROM子句的子查询)

3、table：显示这一行的数据是关于哪张表的

4、type：这列最重要，显示了连接使用了哪种类别,有无使用索引，是使用Explain命令分析性能瓶颈的关键项之一。

    结果值从好到坏依次是：

    system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL

    一般来说，得保证查询至少达到range级别，最好能达到ref，否则就可能会出现性能问题。

5、possible_keys：列指出MySQL能使用哪个索引在该表中找到行

6、key：显示MySQL实际决定使用的键（索引）。如果没有选择索引，键是NULL

7、key_len：显示MySQL决定使用的键长度。如果键是NULL，则长度为NULL。使用的索引的长度。在不损失精确性的情况下，长度越短越好

8、ref：显示使用哪个列或常数与key一起从表中选择行。

9、rows：显示MySQL认为它执行查询时必须检查的行数。

10、Extra：包含MySQL解决查询的详细信息，也是关键参考项之一。

	Distinct  -- 一旦MYSQL找到了与行相联合匹配的行，就不再搜索了 
	Not exists  -- MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行， 就不再搜索了 
	Range checked for each Record（index map:#）  -- 没有找到理想的索引，因此对于从前面表中来的每一个行组合，MYSQL检查使用哪个索引，并用它来从表中返回行。这是使用索引的最慢的连接之一 
	Using filesort -- 看到这个的时候，查询就需要优化了。MYSQL需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行 
	Using index  -- 列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候 
	Using temporary -- 看到这个的时候，查询需要优化了。这里，MYSQL需要创建一个临时表来存储结果，这通常发生在对不同的列集进行ORDER BY上，而不是GROUP BY上 
	Where used -- 使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。如果不想返回表中的全部行，并且连接类型ALL或index，这就会发生，或者是查询有问题
可见，一旦 extra 中的信息是 `Using filesort` 或者 `Using temporary` 时，这时 MySQL 是不能使用索引的，检索会变得很慢。

## 使用 profile 查看语句执行过程中每一个阶段的耗时
查看 profile 是否打开

	select @@profiling;
结果为0表示关闭，结果为1表示打开，打开 profile

	set profiling=1;
然后查询所有执行过的 sql 语句的耗时时间信息

	show profiles;
query_id 是执行的 sql 语句的编号，通过该 id 可以查看某一条 sql 的具体时间信息

	show profile for query [query_id];
通过上面这条命令，查看该sql耗时的具体阶段是在什么时候，上面的命令是按照执行顺序排序的，要查看时间，一般关注exectuing和sending data。然后对这些特定的阶段进行针对性的优化。如果上面这个命令不好对时间排序，可以直接查询数据库的表的方式进行查看。

```sql
set @query_id = 1;
select STATE, SUM(DURATION) as Total_R,
ROUND(
    100 * SUM(DURATION) /
    (select SUM(DURATION)
     from information_schema.profiling
     where query_id = @query_id
     ), 2) as Pct_R,
     COUNT(*) as Calls, 
     SUM(DURATION) / COUNT(*) as "R/Call"
from information_schema.profiling
where query_id = @query_id
group by STATE
order by Total_R desc;
```

## 确定问题并采取的相应的优化措施

常用的优化措施就是添加索引。添加索引能够提升查询效率，但是却降低了插入、更新、删除的性能。

MySQL中添加索引的方法如下：

	alter table tb_name add index index_name (column, ...);
	create index index_name on tb_name (column, ...);
添加主键索引

	alter table tb_name add primary key;

删除索引

	alter table tb_name drop primary key;
	alter table tb_name drop index index_name;
	drop index index_name on tb_name;

查询索引

	show keys from tb_name;
	show index from tb_name;

添加索引只是最基本的方法，对 sql 的详细优化，请参看博文
[mysql sql 百万级数据库优化方案](http://www.cnblogs.com/huangye-dream/archive/2013/05/21/3091906.html)