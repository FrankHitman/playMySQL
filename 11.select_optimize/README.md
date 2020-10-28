# select optimize

## 条件化简
- 移除不必要的括號
- 常量传递（constant_propagation）
- 等值传递（equality_propagation）
- 移除没用的条件（trivial_condition_removal）
- 表达式计算
- HAVING子句和WHERE子句的合并。

如果查询语句中没有出现诸如SUM、MAX等等的聚集函数以及GROUP BY子句，优化器就把HAVING子句和WHERE子句合并起来。
- 常量表检测
- 外连接轉化為內連接

我们把这种在外连接查询中，指定的WHERE子句中包含被驱动表中的列不为NULL值的条件称之为空值拒绝（英文名：reject-NULL）。
在被驱动表的WHERE子句符合空值拒绝的条件后，外连接和内连接可以相互转换。
这种转换带来的好处就是查询优化器可以通过评估表的不同连接顺序的成本，选出成本最低的那种连接顺序来执行查询。

## 子查询优化
### 按返回的结果集区分子查询
- 标量子查询

```SELECT * FROM t1 WHERE m1 = (SELECT MIN(m2) FROM t2);```
- 行子查询
  
```SELECT * FROM t1 WHERE (m1, n1) = (SELECT m2, n2 FROM t2 LIMIT 1);```
- 列子查询

```SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2);```
- 表子查询

```SELECT * FROM t1 WHERE (m1, n1) IN (SELECT m2, n2 FROM t2);```

### 按与外层查询关系来区分子查询
- 不相关子查询
- 相关子查询

```SELECT * FROM t1 WHERE t1.m1 IN (SELECT t2.m2 FROM t2 WHERE t1.n1 = t2.n2);```

### 子查询在布尔表达式中的使用
- ANY/SOME（ANY和SOME是同义词）
```shell script
SELECT * FROM t1 WHERE m1 > ANY(SELECT m2 FROM t2);
# 化簡為
SELECT * FROM t1 WHERE m1 > (SELECT MIN(m2) FROM t2);
```
- ALL
```shell script
SELECT * FROM t1 WHERE m1 > ALL(SELECT m2 FROM t2);
#化簡為
SELECT * FROM t1 WHERE m1 > (SELECT MAX(m2) FROM t2);
```

- EXISTS子查询
```
SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2);
# 只要(SELECT 1 FROM t2)这个查询中有记录，那么整个EXISTS表达式的结果就为TRUE
```

### 子查询语法注意事项
- 子查询必须用小括号扩起来。
- 在SELECT子句中的子查询必须是标量子查询。
- 在想要得到标量子查询或者行子查询，但又不能保证子查询的结果集只有一条记录时，应该使用LIMIT 1语句来限制记录数量。
- 对于[NOT] IN/ANY/SOME/ALL子查询来说，子查询中不允许有LIMIT语句。
- 自查詢不允許有： order by子句，distinct子句，没有聚集函数以及HAVING子句的GROUP BY子句。  
- 不允许在一条语句中增删改某个表的记录时同时还对该表进行子查询。
```shell script
mysql> DELETE FROM t1 WHERE m1 < (SELECT MAX(m1) FROM t1);
ERROR 1093 (HY000): You can't specify target table 't1' for update in FROM clause
```

### 子查询在MySQL中是怎么执行的
#### 标量子查询、行子查询的执行方式
对于包含不相关的标量子查询或者行子查询的查询语句来说，MySQL会分别独立的执行外层查询和子查询，就当作两个单表查询就好了。

对于相关的标量子查询或者行子查询

```SELECT * FROM s1 WHERE key1 = (SELECT common_field FROM s2 WHERE s1.key3 = s2.key3 LIMIT 1);```
- 先从外层查询中获取一条记录，本例中也就是先从s1表中获取一条记录。
- 然后从上一步骤中获取的那条记录中找出子查询中涉及到的值，本例中就是从s1表中获取的那条记录中找出s1.key3列的值，然后执行子查询。
- 最后根据子查询的查询结果来检测外层查询WHERE子句的条件是否成立，如果成立，就把外层查询的那条记录加入到结果集，否则就丢弃。
- 再次执行第一步，获取第二条外层查询中的记录，依次类推～


### IN子查询优化
不直接将不相关子查询的结果集当作外层查询的参数，而是将该结果集写入一个临时表里。写入临时表的过程是这样的：

- 该临时表的列就是子查询结果集中的列。
- 写入临时表的记录会被去重。
- 一般情况下子查询结果集不会大的离谱，所以会为它建立基于内存的使用Memory存储引擎的临时表，而且会为该表建立哈希索引。

如果子查询的结果集非常大，超过了系统变量tmp_table_size或者max_heap_table_size，
临时表会转而使用基于磁盘的存储引擎来保存结果集中的记录，索引类型也对应转变为B+树索引。
```shell script
mysql> show variables like 'tmp_table_size';
+----------------+----------+
| Variable_name  | Value    |
+----------------+----------+
| tmp_table_size | 16777216 |
+----------------+----------+
1 row in set (0.00 sec)

mysql> show variables like 'max_heap_table_size';
+---------------------+----------+
| Variable_name       | Value    |
+---------------------+----------+
| max_heap_table_size | 16777216 |
+---------------------+----------+
1 row in set (0.00 sec)
```
这个将子查询结果集中的记录保存到临时表的过程称之为物化（英文名：Materialize）。为了方便起见，我们就把那个存储子查询结果集的临时表称之为物化表。
正因为物化表中的记录都建立了索引（基于内存的物化表有哈希索引，基于磁盘的有B+树索引），
通过索引执行IN语句判断某个操作数在不在子查询结果集中变得非常快，从而提升了子查询语句的性能

#### 物化表转连接
```shell script
SELECT * FROM s1 
    WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a');

轉化為內連接

SELECT s1.* FROM s1 INNER JOIN materialized_table ON key1 = m_val;
```

##### 将子查询转换为semi-join
```shell script
SELECT * FROM s1 
    WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a');

轉化為

SELECT s1.* FROM s1 INNER JOIN s2 
    ON s1.key1 = s2.common_field 
    WHERE s2.key3 = 'a';
```
在这里提出了一个新概念 --- 半连接（英文名：semi-join）。将s1表和s2表进行半连接的意思就是：
对于s1表的某条记录来说，我们只关心在s2表中是否存在与之匹配的记录，而不关心具体有多少条记录与之匹配，最终的结果集中只保留s1表的记录。
```shell script
SELECT s1.* FROM s1 SEMI JOIN s2
    ON s1.key1 = s2.common_field
    WHERE key3 = 'a';
```
 semi-join只是在MySQL内部采用的一种执行子查询的方式，MySQL并没有提供面向用户的semi-join语法

怎么实现这种所谓的半连接呢？
- Table pullout （子查询中的表上拉）
- DuplicateWeedout execution strategy （重复值消除）
- LooseScan execution strategy （松散扫描）
- Semi-join Materialization execution strategy
- FirstMatch execution strategy （首次匹配）

##### semi-join的适用条件
```shell script
SELECT ... FROM outer_tables 
    WHERE expr IN (SELECT ... FROM inner_tables ...) AND ...;

SELECT ... FROM outer_tables 
    WHERE (oe1, oe2, ...) IN (SELECT ie1, ie2, ... FROM inner_tables ...) AND ...
```

##### 不适用于semi-join的情况
```shell script
SELECT * FROM s1 
    WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a')
        OR key2 > 100;

SELECT * FROM s1 
    WHERE key1 NOT IN (SELECT common_field FROM s2 WHERE key3 = 'a')

SELECT key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a') FROM s1 ;

SELECT * FROM s1 
    WHERE key2 IN (SELECT COUNT(*) FROM s2 GROUP BY key1);

SELECT * FROM s1 WHERE key1 IN (
    SELECT common_field FROM s2 WHERE key3 = 'a' 
    UNION
    SELECT common_field FROM s2 WHERE key3 = 'b'
);

```

##### MySQL仍然留了两手绝活来优化不能转为semi-join查询的子查询
- 对于不相关子查询来说，可以尝试把它们物化之后再参与查询
```shell script
SELECT * FROM s1 
    WHERE key1 NOT IN (SELECT common_field FROM s2 WHERE key3 = 'a')
```

- 不管子查询是相关的还是不相关的，都可以把IN子查询尝试转为EXISTS子查询
````
outer_expr IN (SELECT inner_expr FROM ... WHERE subquery_where)

可以被转换为：

EXISTS (SELECT inner_expr FROM ... WHERE subquery_where AND outer_expr=inner_expr)
````
为啥要转换呢？这是因为不转换的话可能用不到索引，比方说下边这个查询：
````
SELECT * FROM s1
    WHERE key1 IN (SELECT key3 FROM s2 where s1.common_field = s2.common_field) 
        OR key2 > 1000;

SELECT * FROM s1
    WHERE EXISTS (SELECT 1 FROM s2 where s1.common_field = s2.common_field AND s2.key3 = s1.key1) 
        OR key2 > 1000;

# 转为EXISTS子查询时便可以使用到s2表的idx_key3索引了。
````

如果IN子查询不满足转换为semi-join的条件，又不能转换为物化表或者转换为物化表的成本太大，那么它就会被转换为EXISTS查询。


