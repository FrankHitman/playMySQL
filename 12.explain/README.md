# explain

## 執行計劃
一条查询语句在经过MySQL查询优化器的各种基于成本和规则的优化会后生成一个所谓的执行计划，
这个执行计划展示了接下来具体执行查询的方式，比如多表连接的顺序是什么，对于每个表采用什么访问方法来具体执行查询等等

```shell script
# mariadb 可以使用索引
mysql> explain select * from charger order by created_at limit 10;
+------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
| id   | select_type | table   | type  | possible_keys | key            | key_len | ref  | rows | Extra |
+------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
|    1 | SIMPLE      | charger | index | NULL          | idx_created_at | 8       | NULL | 1    |       |
+------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
1 row in set (0.00 sec)

# mysql5.7 居然使用文件排序
mysql> explain select * from siemens_evse_cloud.charger order by created_at limit 10;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | charger | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.01 sec)
```

## explain結果中各個列的意義

列名|	描述
----|----
id	|在一个大的查询语句中每个SELECT关键字都对应一个唯一的id
select_type|	SELECT关键字对应的那个查询的类型
table	|表名
partitions|	匹配的分区信息
type	|针对单表的访问方法
possible_keys|	可能用到的索引
key	|实际上使用的索引
key_len|	实际使用到的索引长度
ref	|当使用索引列等值查询时，与索引列进行等值匹配的对象信息
rows|	预估的需要读取的记录条数
filtered|	某个表经过搜索条件过滤后剩余记录条数的百分比
Extra	|一些额外的信息

### id列
````
mysql> explain select * from charger inner join private_charger on charger.id = private_charger.charger_id;
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+-------+
| id | select_type | table           | partitions | type | possible_keys                                 | key                                           | key_len | ref                           | rows | filtered | Extra |
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+-------+
|  1 | SIMPLE      | charger         | NULL       | ALL  | PRIMARY                                       | NULL                                          | NULL    | NULL                          |    1 |   100.00 | NULL  |
|  1 | SIMPLE      | private_charger | NULL       | ref  | private_charger_charger_id_charger_id_foreign | private_charger_charger_id_charger_id_foreign | 110     | siemens_evse_cloud.charger.id |    1 |   100.00 | NULL  |
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
````
查询语句中每出现一个SELECT关键字，设计MySQL的大叔就会为它分配一个唯一的id值

对于连接查询来说，一个SELECT关键字后边的FROM子句中可以跟随多个表，所以在连接查询的执行计划中，每个表都会对应一条记录，
但是这些记录的id值都是相同的。出现在前边的表表示驱动表，出现在后边的表表示被驱动表。

```shell script
mysql> explain select * from charger where id in (select charger_id from private_charger where is_owner =1);
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+----------------------------------+
| id | select_type | table           | partitions | type | possible_keys                                 | key                                           | key_len | ref                           | rows | filtered | Extra                            |
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+----------------------------------+
|  1 | SIMPLE      | charger         | NULL       | ALL  | PRIMARY                                       | NULL                                          | NULL    | NULL                          |    1 |   100.00 | NULL                             |
|  1 | SIMPLE      | private_charger | NULL       | ref  | private_charger_charger_id_charger_id_foreign | private_charger_charger_id_charger_id_foreign | 110     | siemens_evse_cloud.charger.id |    1 |   100.00 | Using where; FirstMatch(charger) |
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+----------------------------------+
2 rows in set, 1 warning (0.00 sec)

mysql> explain select * from charger inner join private_charger on charger.id = private_charger.charger_id where private_charger.is_owner=1;
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+-------------+
| id | select_type | table           | partitions | type | possible_keys                                 | key                                           | key_len | ref                           | rows | filtered | Extra       |
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+-------------+
|  1 | SIMPLE      | charger         | NULL       | ALL  | PRIMARY                                       | NULL                                          | NULL    | NULL                          |    1 |   100.00 | NULL        |
|  1 | SIMPLE      | private_charger | NULL       | ref  | private_charger_charger_id_charger_id_foreign | private_charger_charger_id_charger_id_foreign | 110     | siemens_evse_cloud.charger.id |    1 |   100.00 | Using where |
+----+-------------+-----------------+------------+------+-----------------------------------------------+-----------------------------------------------+---------+-------------------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

```

对于包含子查询的查询语句来说，就可能涉及多个SELECT关键字，所以在包含子查询的查询语句的执行计划中，每个SELECT关键字都会对应一个唯一的id值
````
CREATE TABLE s1 (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    UNIQUE KEY idx_key2 (key2),
    KEY idx_key3 (key3),
    KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;


CREATE TABLE s2 (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    UNIQUE KEY idx_key2 (key2),
    KEY idx_key3 (key3),
    KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;

mysql> 
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2) OR key3 = 'a';
+----+--------------------+-------+------------+----------------+---------------+----------+---------+------+------+----------+-------------+
| id | select_type        | table | partitions | type           | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+-------+------------+----------------+---------------+----------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | s1    | NULL       | ALL            | idx_key3      | NULL     | NULL    | NULL |    1 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | s2    | NULL       | index_subquery | idx_key1      | idx_key1 | 303     | func |    1 |   100.00 | Using index |
+----+--------------------+-------+------------+----------------+---------------+----------+---------+------+------+----------+-------------+
2 rows in set, 1 warning (0.02 sec)
````

查询优化器可能对涉及子查询的查询语句进行重写，从而转换为连接查询
````
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key3 FROM s2 WHERE common_field = 'a');
+----+-------------+-------+------------+------+---------------+----------+---------+--------------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref          | rows | filtered | Extra                       |
+----+-------------+-------+------------+------+---------------+----------+---------+--------------+------+----------+-----------------------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | idx_key1      | NULL     | NULL    | NULL         |    1 |   100.00 | Using where                 |
|  1 | SIMPLE      | s2    | NULL       | ref  | idx_key3      | idx_key3 | 303     | test.s1.key1 |    1 |   100.00 | Using where; FirstMatch(s1) |
+----+-------------+-------+------------+------+---------------+----------+---------+--------------+------+----------+-----------------------------+
2 rows in set, 1 warning (0.01 sec)

# 但是执行计划中s1和s2表对应的记录的id值全部是1，这就表明了查询优化器将子查询转换为了连接查询
````

对于包含UNION子句的查询语句来说，每个SELECT关键字对应一个id值也是没错的
```shell script
mysql> EXPLAIN SELECT * FROM s1  UNION SELECT * FROM s2;
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
| id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
|  1 | PRIMARY      | s1         | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL            |
|  2 | UNION        | s2         | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
3 rows in set, 1 warning (0.00 sec)
```
UNION子句是干嘛用的，它会把多个查询的结果集合并起来并对结果集中的记录进行去重，怎么去重呢？MySQL使用的是内部的临时表.
id为NULL表明这个临时表是为了合并两个查询的结果集而创建的。

UNION ALL 不需要為結果集去重
```shell script
mysql> EXPLAIN SELECT * FROM s1  UNION ALL SELECT * FROM s2;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | PRIMARY     | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
|  2 | UNION       | s2    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
2 rows in set, 1 warning (0.01 sec)
```

### select_type 列
可以的值

名称|	描述
----|----
SIMPLE	|Simple SELECT (not using UNION or subqueries)
PRIMARY	|Outermost SELECT
UNION	|Second or later SELECT statement in a UNION
UNION RESULT	|Result of a UNION
SUBQUERY	|First SELECT in subquery
DEPENDENT SUBQUERY|	First SELECT in subquery, dependent on outer query
DEPENDENT UNION	|Second or later SELECT statement in a UNION, dependent on outer query
DERIVED	|Derived table
MATERIALIZED|	Materialized subquery
UNCACHEABLE SUBQUERY|	A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query
UNCACHEABLE UNION	|The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY)

- SIMPLW

查询语句中不包含UNION或者子查询的查询都算作是SIMPLE类型

- UNION RESULT

MySQL选择使用临时表来完成UNION查询的去重工作

- SUBQUERY

如果包含子查询的查询语句不能够转为对应的semi-join的形式，并且该子查询是不相关子查询，
并且查询优化器决定采用将该子查询物化的方案来执行该子查询时
```shell script
# 可能是由於表中無數據吧，由如下可以看出使用的是"DEPENDENT SUBQUERY"，而不是"SUBQUERY"。這兒與教程不同。
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2) OR key3 = 'a';
+----+--------------------+-------+------------+----------------+---------------+----------+---------+------+------+----------+-------------+
| id | select_type        | table | partitions | type           | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+-------+------------+----------------+---------------+----------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | s1    | NULL       | ALL            | idx_key3      | NULL     | NULL    | NULL |    1 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | s2    | NULL       | index_subquery | idx_key1      | idx_key1 | 303     | func |    1 |   100.00 | Using index |
+----+--------------------+-------+------------+----------------+---------------+----------+---------+------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```
- DEPENDENT SUBQUERY

如果包含子查询的查询语句不能够转为对应的semi-join的形式，并且该子查询是相关子查询，
则该子查询的第一个SELECT关键字代表的那个查询的select_type就是DEPENDENT SUBQUERY
```shell script
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE s1.key2 = s2.key2) OR key3 = 'a';
+----+--------------------+-------+------------+------+-------------------+----------+---------+--------------+------+----------+-------------+
| id | select_type        | table | partitions | type | possible_keys     | key      | key_len | ref          | rows | filtered | Extra       |
+----+--------------------+-------+------------+------+-------------------+----------+---------+--------------+------+----------+-------------+
|  1 | PRIMARY            | s1    | NULL       | ALL  | idx_key3          | NULL     | NULL    | NULL         |    1 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | s2    | NULL       | ref  | idx_key2,idx_key1 | idx_key2 | 5       | test.s1.key2 |    1 |   100.00 | Using where |
+----+--------------------+-------+------------+------+-------------------+----------+---------+--------------+------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)
```
- DEPENDENT UNION

在包含UNION或者UNION ALL的大查询中，如果各个小查询都依赖于外层查询的话，
那除了最左边的那个小查询之外，其余的小查询的select_type的值就是DEPENDENT UNION
```shell script
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE key1 = 'a' UNION SELECT key1 FROM s1 WHERE key1 = 'b');
+----+--------------------+------------+------------+------+---------------+----------+---------+-------+------+----------+--------------------------+
| id | select_type        | table      | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra                    |
+----+--------------------+------------+------------+------+---------------+----------+---------+-------+------+----------+--------------------------+
|  1 | PRIMARY            | s1         | NULL       | ALL  | NULL          | NULL     | NULL    | NULL  |    1 |   100.00 | Using where              |
|  2 | DEPENDENT SUBQUERY | s2         | NULL       | ref  | idx_key1      | idx_key1 | 303     | const |    1 |   100.00 | Using where; Using index |
|  3 | DEPENDENT UNION    | s1         | NULL       | ref  | idx_key1      | idx_key1 | 303     | const |    1 |   100.00 | Using where; Using index |
| NULL | UNION RESULT       | <union2,3> | NULL       | ALL  | NULL          | NULL     | NULL    | NULL  | NULL |     NULL | Using temporary          |
+----+--------------------+------------+------------+------+---------------+----------+---------+-------+------+----------+--------------------------+
4 rows in set, 1 warning (0.05 sec)
```

- DERIVED

对于采用物化的方式执行的包含派生表的查询，该派生表对应的子查询的select_type就是DERIVED
```shell script
mysql> EXPLAIN SELECT * FROM (SELECT key1, count(*) as c FROM s1 GROUP BY key1) AS derived_s1 where c > 1;
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL   | NULL          | NULL     | NULL    | NULL |    2 |    50.00 | Using where |
|  2 | DERIVED     | s1         | NULL       | index | idx_key1      | idx_key1 | 303     | NULL |    1 |   100.00 | Using index |
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```

- MATERIALIZED

当查询优化器在执行包含子查询的语句时，选择将子查询物化之后与外层查询进行连接查询时，该子查询对应的select_type属性就是MATERIALIZED
```shell script
# 這一條返回也與教程不一致-沒有出現materialized，也可能與沒有數據有關係
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2);
+----+-------------+-------+------------+------+---------------+----------+---------+--------------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref          | rows | filtered | Extra                       |
+----+-------------+-------+------------+------+---------------+----------+---------+--------------+------+----------+-----------------------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | idx_key1      | NULL     | NULL    | NULL         |    1 |   100.00 | Using where                 |
|  1 | SIMPLE      | s2    | NULL       | ref  | idx_key1      | idx_key1 | 303     | test.s1.key1 |    1 |   100.00 | Using index; FirstMatch(s1) |
+----+-------------+-------+------------+------+---------------+----------+---------+--------------+------+----------+-----------------------------+
2 rows in set, 1 warning (0.00 sec)
```

### type 列
- system，
```shell script
mysql> CREATE TABLE t(i int) Engine=MyISAM;
Query OK, 0 rows affected (0.04 sec)

mysql> INSERT INTO t VALUES(1);
Query OK, 1 row affected (0.02 sec)

mysql> explain select * from t;
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | t     | NULL       | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)

mysql> CREATE TABLE tt(i int) Engine=InnoDB;
Query OK, 0 rows affected (0.03 sec)

mysql> insert into tt values(1);
Query OK, 1 row affected (0.01 sec)

mysql> explain select * from tt;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tt    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
- const，
```shell script
mysql> EXPLAIN SELECT * FROM s1 WHERE id = 1;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | s1    | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)
```
- ref

当通过普通的二级索引列与常量进行等值匹配时来查询某个表，那么对该表的访问方法就可能是ref
- eq_ref，
```shell script
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;
+----+-------------+-------+------------+--------+---------------+---------+---------+------------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref        | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+------------+------+----------+-------+
|  1 | SIMPLE      | s1    | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL       |    1 |   100.00 | NULL  |
|  1 | SIMPLE      | s2    | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | test.s1.id |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
```
- fulltext，全文索引
- ref_or_null，

当对普通二级索引进行等值匹配查询，该索引列的值也可以是NULL值时
```shell script
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key1 IS NULL;
+----+-------------+-------+------------+-------------+---------------+----------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type        | possible_keys | key      | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------------+---------------+----------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | s1    | NULL       | ref_or_null | idx_key1      | idx_key1 | 303     | const |    9 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------------+---------------+----------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.01 sec)
# acturally
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key1 IS NULL;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | idx_key1      | NULL | NULL    | NULL |    3 |    66.67 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

```
- index_merge，索引合并

在某些场景下可以使用Intersection、Union、Sort-Union这三种索引合并的方式来执行查询，
```shell script
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key3 = 'a';
+----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+---------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys     | key               | key_len | ref  | rows | filtered | Extra                                       |
+----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+---------------------------------------------+
|  1 | SIMPLE      | s1    | NULL       | index_merge | idx_key1,idx_key3 | idx_key1,idx_key3 | 303,303 | NULL |   14 |   100.00 | Using union(idx_key1,idx_key3); Using where |
+----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+---------------------------------------------+
1 row in set, 1 warning (0.01 sec)
# acturally
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' OR key3 = 'a';
+----+-------------+-------+------------+------+-------------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys     | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+-------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | idx_key1,idx_key3 | NULL | NULL    | NULL |    3 |    55.56 | Using where |
+----+-------------+-------+------------+------+-------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.01 sec)

```
- unique_subquery，

类似于两表连接中被驱动表的eq_ref访问方法，unique_subquery是针对在一些包含IN子查询的查询语句中，
如果查询优化器决定将IN子查询转换为EXISTS子查询，而且子查询可以使用到主键进行等值匹配的话
```shell script
mysql> EXPLAIN SELECT * FROM s1 WHERE key2 IN (SELECT id FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
+----+--------------------+-------+------------+-----------------+------------------+---------+---------+------+------+----------+-------------+
| id | select_type        | table | partitions | type            | possible_keys    | key     | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+-------+------------+-----------------+------------------+---------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | s1    | NULL       | ALL             | idx_key3         | NULL    | NULL    | NULL |    3 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | s2    | NULL       | unique_subquery | PRIMARY,idx_key1 | PRIMARY | 4       | func |    1 |   100.00 | Using where |
+----+--------------------+-------+------------+-----------------+------------------+---------+---------+------+------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)


```

- index_subquery， 子查詢使用普通索引
- range，
```shell script
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN ('a', 'b', 'c');
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | idx_key1      | NULL | NULL    | NULL |    3 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.01 sec)

mysql> EXPLAIN SELECT * FROM s1 WHERE key1 > 'a' AND key1 < 'b';
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | s1    | NULL       | range | idx_key1      | idx_key1 | 303     | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

```
- index，
```shell script
mysql> EXPLAIN SELECT key_part2 FROM s1 WHERE key_part3 = 'a';
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
| id | select_type | table | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | s1    | NULL       | index | NULL          | idx_key_part | 909     | NULL |    3 |    33.33 | Using where; Using index |
+----+-------------+-------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```
- ALL 全表扫描

可能是因為數據少的原因，上面有一些執行計劃使用的是all

### possible_keys列 和 key列
possible_keys列表示在某个查询语句中，对某个表执行单表查询时可能用到的索引有哪些，key列表示实际用到的索引有哪些

possible_keys列中的值并不是越多越好，可能使用的索引越多，查询优化器计算查询成本时就得花费更长时间，
所以如果可以的话，尽量删除那些用不到的索引。

### key_len列
單位是字節

key_len列表示当优化器决定使用某个索引执行查询时，该索引记录的最大长度，它是由这三个部分构成的：

- 对于使用固定长度类型的索引列来说，它实际占用的存储空间的最大长度就是该固定值，对于指定字符集的变长类型的索引列来说，比如某个索引列的类型是VARCHAR(100)，使用的字符集是utf8，那么该列实际占用的最大存储空间就是100 × 3 = 300个字节。
- 如果该索引列可以存储NULL值，则key_len比不可以存储NULL值时多1个字节。
- 对于变长字段来说，都会有2个字节的空间来存储该变长列的实际长度。

### ref列
当使用索引列等值匹配的条件去执行查询时，也就是在访问方法是const、eq_ref、ref、ref_or_null、unique_subquery、index_subquery其中之一时，
ref列展示的就是与索引列作等值匹配的东东是个啥，比如只是一个常数或者是某个列

### rows列
如果查询优化器决定使用全表扫描的方式对某个表执行查询时，执行计划的rows列就代表预计需要扫描的行数，
如果使用索引来执行查询时，执行计划的rows列就代表预计扫描的索引记录行数。

### filtered列
單位是%

之前在分析连接查询的成本时提出过一个condition filtering的概念，就是MySQL在计算驱动表扇出时采用的一个策略：

```shell script
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key1 WHERE s1.common_field = 'a';
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref               | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | idx_key1      | NULL     | NULL    | NULL              | 9688 |    10.00 | Using where |
|  1 | SIMPLE      | s2    | NULL       | ref  | idx_key1      | idx_key1 | 303     | xiaohaizi.s1.key1 |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```
从执行计划中可以看出来，查询优化器打算把s1当作驱动表，s2当作被驱动表。我们可以看到驱动表s1表的执行计划的rows列为9688， filtered列为10.00，
这意味着驱动表s1的扇出值就是9688 × 10.00% = 968.8，这说明还要对被驱动表执行大约968次查询。

### Extra列
Extra列是用来说明一些额外信息的
- Using index

当我们的查询列表以及搜索条件中只包含属于某个索引的列，也就是在可以使用索引覆盖的情况下，在Extra列将会提示该额外信息。
比方说下边这个查询中只需要用到idx_key1而不需要回表操作：

- Using index condition
```shell script
mysql> EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND key1 LIKE '%b';
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | s1    | NULL       | range | idx_key1      | idx_key1 | 303     | NULL |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.01 sec)

```
1. 先根据key1 > 'z'这个条件，定位到二级索引idx_key1中对应的二级索引记录。

2. 对于指定的二级索引记录，先不着急回表，而是先检测一下该记录是否满足key1 LIKE '%a'这个条件，
如果这个条件不满足，则该二级索引记录压根儿就没必要回表。

3. 对于满足key1 LIKE '%a'这个条件的二级索引记录执行回表操作。

把这个改进称之为索引条件下推（英文名：Index Condition Pushdown）。

- Using where

当我们使用全表扫描来执行对某个表的查询，并且该语句的WHERE子句中有针对该表的搜索条件时，在Extra列中会提示上述额外信息

- Using join buffer (Block Nested Loop)

在连接查询执行过程中，当被驱动表不能有效的利用索引加快访问速度，
MySQL一般会为其分配一块名叫join buffer的内存块来加快查询速度，也就是我们所讲的基于块的嵌套循环算法
```shell script
mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.common_field = s2.common_field;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | s2    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL                                               |
|  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

```

1. Using join buffer (Block Nested Loop)：这是因为对表s2的访问不能有效利用索引，只好退而求其次，使用join buffer来减少对s2表的访问次数，从而提高性能。

2. Using where：可以看到查询语句中有一个s1.common_field = s2.common_field条件，因为s1是驱动表，s2是被驱动表，
所以在访问s2表时，s1.common_field的值已经确定下来了，所以实际上查询s2表的条件就是s2.common_field = 一个常数，所以提示了Using where额外信息。

- Using filesort

很多情况下排序操作无法使用到索引，只能在内存中（记录较少的时候）或者磁盘中（记录较多的时候）进行排序，设计MySQL的大叔把这种在内存中或者磁盘上进行排序的方式统称为文件排序（英文名：filesort）

- Using temporary

在许多查询的执行过程中，MySQL可能会借助临时表来完成一些功能，比如去重、排序之类的，比如我们在执行许多包含DISTINCT、GROUP BY、UNION等子句的查询过程中
```shell script
mysql> EXPLAIN SELECT common_field, COUNT(*) AS amount FROM s1 GROUP BY common_field;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | Using temporary; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)

```
这是因为MySQL会在包含GROUP BY子句的查询中默认添加上ORDER BY子句，也就是说上述查询其实和下边这个查询等价：

    EXPLAIN SELECT common_field, COUNT(*) AS amount FROM s1 GROUP BY common_field ORDER BY common_field;

````shell script
mysql>  EXPLAIN SELECT common_field, COUNT(*) AS amount FROM s1 GROUP BY common_field ORDER BY NULL;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | Using temporary |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
1 row in set, 1 warning (0.00 sec)

````

- FirstMatch(tbl_name)

在将In子查询转为semi-join时，如果采用的是FirstMatch执行策略，
则在被驱动表执行计划的Extra列就是显示FirstMatch(tbl_name)提示

## json
```shell script
mysql> EXPLAIN FORMAT=JSON SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key2 WHERE s1.common_field = 'a'\G
*************************** 1. row ***************************

EXPLAIN: {
  "query_block": {
    "select_id": 1,     # 整个查询语句只有1个SELECT关键字，该关键字对应的id号为1
    "cost_info": {
      "query_cost": "3197.16"   # 整个查询的执行成本预计为3197.16
    },
    "nested_loop": [    # 几个表之间采用嵌套循环连接算法执行
    
    # 以下是参与嵌套循环连接算法的各个表的信息
      {
        "table": {
          "table_name": "s1",   # s1表是驱动表
          "access_type": "ALL",     # 访问方法为ALL，意味着使用全表扫描访问
          "possible_keys": [    # 可能使用的索引
            "idx_key1"
          ],
          "rows_examined_per_scan": 9688,   # 查询一次s1表大致需要扫描9688条记录
          "rows_produced_per_join": 968,    # 驱动表s1的扇出是968
          "filtered": "10.00",  # condition filtering代表的百分比
          "cost_info": {
            "read_cost": "1840.84",     # 稍后解释
            "eval_cost": "193.76",      # 稍后解释
            "prefix_cost": "2034.60",   # 单次查询s1表总共的成本
            "data_read_per_join": "1M"  # 读取的数据量
          },
          "used_columns": [     # 执行查询中涉及到的列
            "id",
            "key1",
            "key2",
            "key3",
            "key_part1",
            "key_part2",
            "key_part3",
            "common_field"
          ],
          
          # 对s1表访问时针对单表查询的条件
          "attached_condition": "((`xiaohaizi`.`s1`.`common_field` = 'a') and (`xiaohaizi`.`s1`.`key1` is not null))"
        }
      },
      {
        "table": {
          "table_name": "s2",   # s2表是被驱动表
          "access_type": "ref",     # 访问方法为ref，意味着使用索引等值匹配的方式访问
          "possible_keys": [    # 可能使用的索引
            "idx_key2"
          ],
          "key": "idx_key2",    # 实际使用的索引
          "used_key_parts": [   # 使用到的索引列
            "key2"
          ],
          "key_length": "5",    # key_len
          "ref": [      # 与key2列进行等值匹配的对象
            "xiaohaizi.s1.key1"
          ],
          "rows_examined_per_scan": 1,  # 查询一次s2表大致需要扫描1条记录
          "rows_produced_per_join": 968,    # 被驱动表s2的扇出是968（由于后边没有多余的表进行连接，所以这个值也没啥用）
          "filtered": "100.00",     # condition filtering代表的百分比
          
          # s2表使用索引进行查询的搜索条件
          "index_condition": "(`xiaohaizi`.`s1`.`key1` = `xiaohaizi`.`s2`.`key2`)",
          "cost_info": {
            "read_cost": "968.80",      # 稍后解释
            "eval_cost": "193.76",      # 稍后解释
            "prefix_cost": "3197.16",   # 单次查询s1、多次查询s2表总共的成本
            "data_read_per_join": "1M"  # 读取的数据量
          },
          "used_columns": [     # 执行查询中涉及到的列
            "id",
            "key1",
            "key2",
            "key3",
            "key_part1",
            "key_part2",
            "key_part3",
            "common_field"
          ]
        }
      }
    ]
  }
}
1 row in set, 2 warnings (0.00 sec)
```
由于s2表是被驱动表，所以可能被读取多次，这里的read_cost和eval_cost是访问多次s2表后累加起来的值，
大家主要关注里边儿的prefix_cost的值代表的是整个连接查询预计的成本，也就是单次查询s1表和多次查询s2表后的成本的和，

    968.80 + 193.76 + 2034.60 = 3197.16

在我们使用EXPLAIN语句查看了某个查询的执行计划后，紧接着还可以使用SHOW WARNINGS语句查看与这个查询的执行计划有关的一些扩展信息
````shell script
mysql> SHOW WARNINGS\G
*************************** 1. row ***************************
  Level: Warning
   Code: 1739
Message: Cannot use ref access on index 'idx_key1' due to type or collation conversion on field 'key1'
*************************** 2. row ***************************
  Level: Warning
   Code: 1739
Message: Cannot use range access on index 'idx_key1' due to type or collation conversion on field 'key1'
*************************** 3. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `test`.`s1`.`id` AS `id`,`test`.`s1`.`key1` AS `key1`,`test`.`s1`.`key2` AS `key2`,`test`.`s1`.`key3` AS `key3`,`test`.`s1`.`key_part1` AS `key_part1`,`test`.`s1`.`key_part2` AS `key_part2`,`test`.`s1`.`key_part3` AS `key_part3`,`test`.`s1`.`common_field` AS `common_field`,`test`.`s2`.`id` AS `id`,`test`.`s2`.`key1` AS `key1`,`test`.`s2`.`key2` AS `key2`,`test`.`s2`.`key3` AS `key3`,`test`.`s2`.`key_part1` AS `key_part1`,`test`.`s2`.`key_part2` AS `key_part2`,`test`.`s2`.`key_part3` AS `key_part3`,`test`.`s2`.`common_field` AS `common_field` from `test`.`s1` join `test`.`s2` where ((`test`.`s1`.`common_field` = 'a') and (`test`.`s1`.`key1` = `test`.`s2`.`key2`))
3 rows in set (0.00 sec)

````
当Code值为1003时，Message字段展示的信息类似于(不是等價於)查询优化器将我们的查询语句重写后的语句。
