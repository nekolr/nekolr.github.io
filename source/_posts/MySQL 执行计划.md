---
title: MySQL 执行计划
date: 2020/9/8 12:50:0
tags: [数据库]
categories: [数据库]
---

一条查询语句在经过 MySQL 的查询优化器分析后，会生成一个所谓的执行计划，这个计划能够告诉我们接下来的查询的一些信息，比如单表采用哪种访问方法，多表连接的顺序，预估需要读取的记录数等等。

<!--more-->

我们平常使用的查询语句，甚至是增删改的语句，都可以在前面加上 `EXPLAIN` 关键字列出该条语句的执行计划，不过一般存在性能瓶颈的都是查询语句，所以我们主要关注的也是查询语句。下面先简单说明一下 EXPLAIN 语句输出的各个列的含义。

列名 | 描述
--|--
id | SELECT 标识符，每个 SELECT 关键字都对应一个唯一的 id
select_type | 查询的类型
table | 表名
partitions | 匹配的分区，不使用分区表时，默认为 NULL
type | 单表访问的方法
possible_keys | 可能使用的索引
key | 实际使用的索引
key_len | 实际使用索引的字段长度
ref | 使用索引列等值查询时，与索引列进行等值匹配的对象，可以是一个常数或者某个列
rows | 预估需要扫描的行数，也就是结果集的行数
filtered | 通过条件过滤出的行数的百分比估值
Extra | 一些额外的信息

# table
不管我们的查询语句有多复杂，最终都会分解成一个个的单表查询，所以 EXPLAIN 输出的每条记录都对应着某个单表的访问，该条记录的 table 列就代表这张表的名称。比如一个简单的查询：

```sql
explain select * from user where id = 12286;

+--+-----------+------+----------+-----+-------------+-------+-------+-----+----+--------+-----+
|id|select_type|table |partitions|type |possible_keys|key    |key_len|ref  |rows|filtered|Extra|
+--+-----------+------+----------+-----+-------------+-------+-------+-----+----+--------+-----+
|1 |SIMPLE     |user  |NULL      |const|PRIMARY      |PRIMARY|4      |const|1   |100     |NULL |
+--+-----------+------+----------+-----+-------------+-------+-------+-----+----+--------+-----+
```

然后再看看一个连接查询：

```sql
explain select * from user t1 inner join user t2;

+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------------+
|id|select_type|table|partitions|type|possible_keys|key |key_len|ref |rows  |filtered|Extra                                |
+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------------+
|1 |SIMPLE     |t1   |NULL      |ALL |NULL         |NULL|NULL   |NULL|991800|100     |NULL                                 |
|1 |SIMPLE     |t2   |NULL      |ALL |NULL         |NULL|NULL   |NULL|991800|100     |Using join buffer (Block Nested Loop)|
+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------------+
```

# id
一般情况下的查询只有一个 SELECT 关键字，比如普通的单表查询或者连接查询，但是有时候也会有多个 SELECT 关键字，比如包含子查询语句以及 UNION 查询等。在整个查询语句中，每出现一个 SELECT 关键字，MySQL 就会为它分配一个唯一的 id。

对于连接查询来说，有一个比较重要的规则就是，在连接查询的执行计划中，每个表都会对应一条记录，它们的 id 值都相同，但是出现在前面的表是驱动表，而出现在后面的表是被驱动表。

对于子查询来说，一般会包含多个 SELECT 关键字，每个关键字都会对应一个唯一的 id 值，比如：

```sql
explain select * from user where id in (select id from user) or username = 'Bob';

+--+-----------+-----+----------+-----+-------------+--------+-------+----+------+--------+-----------+
|id|select_type|table|partitions|type |possible_keys|key     |key_len|ref |rows  |filtered|Extra      |
+--+-----------+-----+----------+-----+-------------+--------+-------+----+------+--------+-----------+
|1 |PRIMARY    |user |NULL      |ALL  |idx_username |NULL    |NULL   |NULL|991800|100     |Using where|
|2 |SUBQUERY   |user |NULL      |index|PRIMARY      |idx_code|5      |NULL|991800|100     |Using index|
+--+-----------+-----+----------+-----+-------------+--------+-------+----+------+--------+-----------+
```

但是有时候查询优化器可能会对涉及子查询的查询语句进行重写并转换为连接查询，比如：

```sql
explain select * from user where id in (select id from user);

+--+-----------+-----+----------+------+-------------+-------+-------+------------+------+--------+-----------+
|id|select_type|table|partitions|type  |possible_keys|key    |key_len|ref         |rows  |filtered|Extra      |
+--+-----------+-----+----------+------+-------------+-------+-------+------------+------+--------+-----------+
|1 |SIMPLE     |user |NULL      |ALL   |PRIMARY      |NULL   |NULL   |NULL        |991800|100     |NULL       |
|1 |SIMPLE     |user |NULL      |eq_ref|PRIMARY      |PRIMARY|4      |demo.user.id|1     |100     |Using index|
+--+-----------+-----+----------+------+-------------+-------+-------+------------+------+--------+-----------+
```

对于 UNION 查询来说，由于还需要去重，所以执行计划可能会有点不一样，比如：

```sql
explain select * from user t1 union select * from user t2;

+----+------------+----------+----------+----+-------------+----+-------+----+------+--------+---------------+
|id  |select_type |table     |partitions|type|possible_keys|key |key_len|ref |rows  |filtered|Extra          |
+----+------------+----------+----------+----+-------------+----+-------+----+------+--------+---------------+
|1   |PRIMARY     |t1        |NULL      |ALL |NULL         |NULL|NULL   |NULL|991800|100     |NULL           |
|2   |UNION       |t2        |NULL      |ALL |NULL         |NULL|NULL   |NULL|991800|100     |NULL           |
|NULL|UNION RESULT|<union1,2>|NULL      |ALL |NULL         |NULL|NULL   |NULL|NULL  |NULL    |Using temporary|
+----+------------+----------+----------+----+-------------+----+-------+----+------+--------+---------------+
```

为了将 id 为 1 的查询和 id 为 2 的查询结果集合并起来并去重，需要创建一个临时表，也就是上面的 `<union1,2>`，而由于 UNION ALL 不需要去重，所以执行计划里没有 id 为 NULL 的记录。

总的来说，一般情况下，id 值相同时，按照从上到下的顺序执行；id 值不同时，id 值大的先执行。

# select_type
名称 | 描述
--|--
SIMPLE | 简单查询，不包含子查询或者 UNION 查询
PRIMARY | 最外层的那个查询
UNION | UNION 后面的查询
UNION RESULT | UNION 去重时使用的临时表的查询
SUBQUERY | 不依赖外部查询的子查询
DEPENDENT SUBQUERY | 依赖外部查询的子查询
DERIVED | 采用物化的方式执行包含派生表的查询
MATERIALIZED | 查询优化器选择将子查询物化后与外层查询进行连接查询时

## SIMPLE
查询中不包含子查询或者 UNION 查询的都算作 SIMPLE 类型的查询，当然，连接查询也是 SIMPLE 类型的，比如：

```sql
explain select * from user t1 inner join user t2;

+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------------+
|id|select_type|table|partitions|type|possible_keys|key |key_len|ref |rows  |filtered|Extra                                |
+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------------+
|1 |SIMPLE     |t1   |NULL      |ALL |NULL         |NULL|NULL   |NULL|991800|100     |NULL                                 |
|1 |SIMPLE     |t2   |NULL      |ALL |NULL         |NULL|NULL   |NULL|991800|100     |Using join buffer (Block Nested Loop)|
+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------------+
```

## PRIMARY
对于包含子查询或者 UNION 查询的大查询来说，它是由几个小的查询组成的，其中最左边也就是最外层的那个小查询就是 PRIMARY 类型的。

## SUBQUERY
如果包含子查询的查询语句不能转换为对应的 semi-join 的形式，并且该子查询不依赖外部的查询，同时查询优化器决定将子查询进行物化，此时子查询的第一个 SELECT 关键字代表的那个查询就是 SUBQUERY 类型的，比如：

```sql
explain select * from user where id in (select id from user) or username = 'Bob';

+--+-----------+-----+----------+-----+-------------+--------+-------+----+------+--------+-----------+
|id|select_type|table|partitions|type |possible_keys|key     |key_len|ref |rows  |filtered|Extra      |
+--+-----------+-----+----------+-----+-------------+--------+-------+----+------+--------+-----------+
|1 |PRIMARY    |user |NULL      |ALL  |idx_username |NULL    |NULL   |NULL|991800|100     |Using where|
|2 |SUBQUERY   |user |NULL      |index|PRIMARY      |idx_code|5      |NULL|991800|100     |Using index|
+--+-----------+-----+----------+-----+-------------+--------+-------+----+------+--------+-----------+
```

**由于 SUBQUERY 类型的子查询会被物化，所以它只需要执行一遍即可。**

## DEPENDENT SUBQUERY
如果包含子查询的查询语句不能转换为对应的 semi-join 的形式，并且该子查询依赖于外部的查询，那么子查询的第一个 SELECT 关键字代表的那个查询就是 DEPENDENT SUBQUERY 类型的，比如：

```sql
explain select * from user t1 where id in (select id from user t2 where t1.id = t2.id) or username = 'Bob';

+--+------------------+-----+----------+------+-------------+-------+-------+----------+------+--------+------------------------+
|id|select_type       |table|partitions|type  |possible_keys|key    |key_len|ref       |rows  |filtered|Extra                   |
+--+------------------+-----+----------+------+-------------+-------+-------+----------+------+--------+------------------------+
|1 |PRIMARY           |t1   |NULL      |ALL   |idx_username |NULL   |NULL   |NULL      |991800|100     |Using where             |
|2 |DEPENDENT SUBQUERY|t2   |NULL      |eq_ref|PRIMARY      |PRIMARY|4      |demo.t1.id|1     |100     |Using where; Using index|
+--+------------------+-----+----------+------+-------------+-------+-------+----------+------+--------+------------------------+
```

**与 SUBQUERY 不同，查询类型为 DEPENDENT SUBQUERY 的查询可能会执行多次。**

## DERIVED
对于采用物化的方式执行的包含派生表的查询，该派生表对应的子查询就是 DERIVED 类型的，比如：

```sql
explain select * from (select province, count(*) as count from user group by province) as derived_t;

+--+-----------+----------+----------+-----+-------------+-----------+-------+----+------+--------+-----------+
|id|select_type|table     |partitions|type |possible_keys|key        |key_len|ref |rows  |filtered|Extra      |
+--+-----------+----------+----------+-----+-------------+-----------+-------+----+------+--------+-----------+
|1 |PRIMARY    |<derived2>|NULL      |ALL  |NULL         |NULL       |NULL   |NULL|991800|100     |NULL       |
|2 |DERIVED    |user      |NULL      |index|idx_address  |idx_address|909    |NULL|991800|100     |Using index|
+--+-----------+----------+----------+-----+-------------+-----------+-------+----+------+--------+-----------+
```

## MATERIALIZED
查询优化器在执行子查询时，如果选择将子查询物化后与外层查询进行连接查询时，该子查询就是 MATERIALIZED 类型的，比如：

```sql
explain select * from user where id in (select user_id from post);

+--+------------+-----------+----------+------+-------------+-------+-------+-------------------+----+--------+-----------+
|id|select_type |table      |partitions|type  |possible_keys|key    |key_len|ref                |rows|filtered|Extra      |
+--+------------+-----------+----------+------+-------------+-------+-------+-------------------+----+--------+-----------+
|1 |SIMPLE      |<subquery2>|NULL      |ALL   |NULL         |NULL   |NULL   |NULL               |NULL|100     |Using where|
|1 |SIMPLE      |user       |NULL      |eq_ref|PRIMARY      |PRIMARY|4      |<subquery2>.user_id|1   |100     |NULL       |
|2 |MATERIALIZED|post       |NULL      |ALL   |NULL         |NULL   |NULL   |NULL               |1   |100     |NULL       |
+--+------------+-----------+----------+------+-------------+-------+-------+-------------------+----+--------+-----------+
```

最后一条记录的 id 为 2，查询类型是 MATERIALIZED，说明查询优化器对它进行了物化，而前两条记录的 id 为 1，说明这两条记录对应的表进行了连接查询，`<subquery2>` 对应的就是物化表。

# type
对应的是单表的访问方法，除了之前讲过的那些，还有一些比较特殊的访问方法，比如 system、eq_ref、fulltext 等。

## system
当表中只有一条记录并且该表使用的存储引擎的统计数据是精确的，比如 MyISAM、Memory 等，那么该表的访问方法就是 system。

## eq_ref
在进行连接查询时，如果被驱动表是通过主键或者唯一二级索引等值匹配的方式进行访问的话，那么对该被驱动表的访问方法就是 eq_ref，比如：

```sql
explain select * from user t1 inner join user t2 on t1.id = t2.id;

+--+-----------+-----+----------+------+-------------+-------+-------+----------+------+--------+-----+
|id|select_type|table|partitions|type  |possible_keys|key    |key_len|ref       |rows  |filtered|Extra|
+--+-----------+-----+----------+------+-------------+-------+-------+----------+------+--------+-----+
|1 |SIMPLE     |t1   |NULL      |ALL   |PRIMARY      |NULL   |NULL   |NULL      |991800|100     |NULL |
|1 |SIMPLE     |t2   |NULL      |eq_ref|PRIMARY      |PRIMARY|4      |demo.t1.id|1     |100     |NULL |
+--+-----------+-----+----------+------+-------------+-------+-------+----------+------+--------+-----+
```

# possible_keys 和 key
possible_keys 表示可能会用到的索引，而 key 代表的是实际用到的索引，比如下面的查询语句可能用到的索引为 ids_code 和 idx_username，但是在计算成本之后，还是选择了 idx_username。

```sql
explain select * from user where code > 1000 and username = 'Bob';

+--+-----------+-----+----------+----+---------------------+------------+-------+-----+----+--------+-----------+
|id|select_type|table|partitions|type|possible_keys        |key         |key_len|ref  |rows|filtered|Extra      |
+--+-----------+-----+----------+----+---------------------+------------+-------+-----+----+--------+-----------+
|1 |SIMPLE     |user |NULL      |ref |idx_code,idx_username|idx_username|303    |const|1724|50      |Using where|
+--+-----------+-----+----------+----+---------------------+------------+-------+-----+----+--------+-----------+
```

需要注意的是，在使用 index 访问方法时，possible_keys 的值为 NULL，但是 key 是有值的。另外 possible_keys 列中的值并不是越多越好，可能使用的索引越多，查询优化器计算各个索引的查询成本时就要花费更多的时间。

# key_len
在一般情况下，key_len 代表当查询优化器决定使用某个索引时，该索引列的最大长度。对于使用固定长度类型的索引列来说，它实际占用的存储空间就是该固定值；对于变长类型的字段来说，比如 VARCHAR(100)，使用 UTF8 字符集，那么它实际占用的最大存储空间就是 100 * 3 = 300 个字节，同时还需要 2 个字节的空间来存储变长字段的实际长度值。如果索引列可以存储 NULL 值，那么 key_len 会比不能为空时多一个字节。比如下面这个例子，code 列是 int 类型的，并且可以存储 NULL 值，所以 key_len 的大小为 5。

```sql
explain select * from user where code  = 122;

+--+-----------+-----+----------+-----+-------------+--------+-------+-----+----+--------+-----+
|id|select_type|table|partitions|type |possible_keys|key     |key_len|ref  |rows|filtered|Extra|
+--+-----------+-----+----------+-----+-------------+--------+-------+-----+----+--------+-----+
|1 |SIMPLE     |user |NULL      |const|idx_code     |idx_code|5      |const|1   |100     |NULL |
+--+-----------+-----+----------+-----+-------------+--------+-------+-----+----+--------+-----+
```

当然有时候，key_len 代表的是实际用到的索引列的长度，比如在使用联合索引时，只用到一个索引列和两个索引列，key_len 的值是不一样的。

```sql
explain select * from user where province = '山东省';

+--+-----------+-----+----------+----+-------------+-----------+-------+-----+-----+--------+-----+
|id|select_type|table|partitions|type|possible_keys|key        |key_len|ref  |rows |filtered|Extra|
+--+-----------+-----+----------+----+-------------+-----------+-------+-----+-----+--------+-----+
|1 |SIMPLE     |user |NULL      |ref |idx_address  |idx_address|303    |const|95924|100     |NULL |
+--+-----------+-----+----------+----+-------------+-----------+-------+-----+-----+--------+-----+
```

```sql
explain select * from user where province = '山东省' and city = '青岛市';

+--+-----------+-----+----------+----+-------------+-----------+-------+-----------+----+--------+-----+
|id|select_type|table|partitions|type|possible_keys|key        |key_len|ref        |rows|filtered|Extra|
+--+-----------+-----+----------+----+-------------+-----------+-------+-----------+----+--------+-----+
|1 |SIMPLE     |user |NULL      |ref |idx_address  |idx_address|606    |const,const|6000|100     |NULL |
+--+-----------+-----+----------+----+-------------+-----------+-------+-----------+----+--------+-----+
```

# filtered
这个还是举例子比较清楚，比如：

```sql
explain select * from user where code < 1000 and username = 'Bob';

+--+-----------+-----+----------+-----+---------------------+--------+-------+----+----+--------+----------------------------------+
|id|select_type|table|partitions|type |possible_keys        |key     |key_len|ref |rows|filtered|Extra                             |
+--+-----------+-----+----------+-----+---------------------+--------+-------+----+----+--------+----------------------------------+
|1 |SIMPLE     |user |NULL      |range|idx_code,idx_username|idx_code|5      |NULL|999 |0.17    |Using index condition; Using where|
+--+-----------+-----+----------+-----+---------------------+--------+-------+----+----+--------+----------------------------------+
```

在这个例子中，通过索引条件 code < 1000，查询优化器估算出了需要扫描的行数为 999，而这 999 行中，经过条件过滤，最终只有一条结果，也就是估算出有 0.17 % 的行满足剩下的条件（`username = 'Bob'`）。

# Extra
Extra 可以提供一些额外的信息，使我们能够更加准确地理解查询优化器到底会如何执行给定的查询语句。

- No tables used
查询语句没有 FROM 子句，比如 SELECT 1，或者 SELECT 1 FROM dual。

- Impossible WHERE
查询语句的 WHERE 子句永远为 FALSE 时，比如 SELECT * FROM user WHERE 1 != 1。

- Using index
当使用覆盖索引时，比如 `SELECT province, city, county FROM user WHERE province = '上海市'`。

- Using index condition
在查询语句的执行过程中使用了**索引条件下推（Index Condition Pushdown）**这个特性。举个例子，比如：
```sql
explain select * from user where username > 'z' and username like '%ce';

+--+-----------+-----+----------+-----+-------------+------------+-------+----+----+--------+---------------------+
|id|select_type|table|partitions|type |possible_keys|key         |key_len|ref |rows|filtered|Extra                |
+--+-----------+-----+----------+-----+-------------+------------+-------+----+----+--------+---------------------+
|1 |SIMPLE     |user |NULL      |range|idx_username |idx_username|303    |NULL|3516|100     |Using index condition|
+--+-----------+-----+----------+-----+-------------+------------+-------+----+----+--------+---------------------+
```
上面的例子中，`username > 'z'` 可以通过索引列使用 range 访问方法，但是 `username like '%ce'` 不行，在以前的 MySQL 中，此时需要回表获取完整的用户记录，然后再检查记录是否符合条件；在 5.6 以后的版本中，此时先不急着回表，而是直接检查是否满足条件，如果没有满足就没必要回表，这样就可以省去好多回表的 I/O 成本。

- Using where
当使用全表扫描的访问方法查询某个表，同时语句中有对应的 where 条件时。或者当使用索引查询，并且该查询的 where 子句中还使用了其他非索引的列作为搜索条件。

- Zero limit
当 limit 子句的参数为 0 时，这意味着根本不打算从表中读出任何记录。

- Using filesort
有时候排序操作无法使用索引，这时只能在内存中（记录较少时）或者磁盘中（记录较多时）进行排序，这种排序方式称为文件排序（filesort）。

- Using temporary
在很多查询中，MySQL 可能需要借助临时表来完成一些功能，比如使用 UNION、DISTINCT 去重、使用 GROUP BY 排序等。
```sql
explain select DISTINCT avatar from user;

+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+---------------+
|id|select_type|table|partitions|type|possible_keys|key |key_len|ref |rows  |filtered|Extra          |
+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+---------------+
|1 |SIMPLE     |user |NULL      |ALL |NULL         |NULL|NULL   |NULL|991800|100     |Using temporary|
+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+---------------+
```
```sql
explain select avatar, count(*) from user group by avatar;

+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------+
|id|select_type|table|partitions|type|possible_keys|key |key_len|ref |rows  |filtered|Extra                          |
+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------+
|1 |SIMPLE     |user |NULL      |ALL |NULL         |NULL|NULL   |NULL|991800|100     |Using temporary; Using filesort|
+--+-----------+-----+----------+----+-------------+----+-------+----+------+--------+-------------------------------+
```
由于 MySQL 默认会在 GROUP BY 子句中添加 ORDER BY 子句，所以这里的分组还出现了 Using filesort，如果我们不想进行排序，可以显式地使用 ORDER BY NULL 来指定。

# 查询成本
通过 EXPLAIN 输出的执行计划缺少了一个衡量执行计划“好坏”的重要因素，那就是**成本**。从 MySQL 5.6 版本开始，我们可以通过在 EXPLAIN 后面加上 FORMAT=JSON 来得到一个 json 格式的执行计划，里面包含了查询成本。比如：

```sql
explain FORMAT=JSON select avatar, count(*) from user group by avatar;

"{
  ""query_block"": {
    ""select_id"": 1,
    ""cost_info"": {
      ""query_cost"": ""1200635.00""
    },
    ""grouping_operation"": {
      ""using_temporary_table"": true,
      ""using_filesort"": true,
      ""cost_info"": {
        ""sort_cost"": ""991800.00""
      },
      ""table"": {
        ""table_name"": ""user"",
        ""access_type"": ""ALL"",
        ""rows_examined_per_scan"": 991800,
        ""rows_produced_per_join"": 991800,
        ""filtered"": ""100.00"",
        ""cost_info"": {
          ""read_cost"": ""10475.00"",
          ""eval_cost"": ""198360.00"",
          ""prefix_cost"": ""208835.00"",
          ""data_read_per_join"": ""1G""
        },
        ""used_columns"": [
          ""id"",
          ""avatar""
        ]
      }
    }
  }
}"
```

这里需要关注的重点就是 cost_info，其中 read_cost 包含 IO 成本以及扫描 `rows * (1 - filtered)` 条记录的 CPU 成本，而 eval_cost 是扫描 rows * filtered 条记录的成本，而 prefix_cost = read_cost + eval_cost。data_read_per_join 表示此次查询中需要读取的数据量。如果觉得这样还是不够详细，可以直接修改 MySQL 的参数来启动查询优化器的追踪。
