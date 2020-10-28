# innodb save

## 两种不同的统计数据存储方式
```
CREATE TABLE 表名 (...) Engine=InnoDB, STATS_PERSISTENT = (1|0);

ALTER TABLE 表名 Engine=InnoDB, STATS_PERSISTENT = (1|0);
```
当STATS_PERSISTENT=1时，表明我们想把该表的统计数据永久的存储到磁盘上，当STATS_PERSISTENT=0时，表明我们想把该表的统计数据临时的存储到内存中。
如果我们在创建表时未指定STATS_PERSISTENT属性，那默认采用系统变量innodb_stats_persistent的值作为该属性的值。
````
mysql> show variables like 'innodb_stats_persistent';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_stats_persistent | ON    |
+-------------------------+-------+
1 row in set (0.02 sec)
````
## 基于磁盘的永久性统计数据
    mysql> show tables from mysql like 'innodb%';
    +---------------------------+
    | Tables_in_mysql (innodb%) |
    +---------------------------+
    | innodb_index_stats        |
    | innodb_table_stats        |
    +---------------------------+
    2 rows in set (0.01 sec)
### innodb_index_stats
    mysql> select * from mysql.innodb_index_stats where table_name like 'position' ;
    +--------------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
    | database_name      | table_name | index_name | last_update         | stat_name    | stat_value | sample_size | stat_description                  |
    +--------------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
    | siemens_evse_cloud | position   | PRIMARY    | 2020-01-22 10:10:58 | n_diff_pfx01 |      46631 |          20 | id                                |
    | siemens_evse_cloud | position   | PRIMARY    | 2020-01-22 10:10:58 | n_leaf_pages |        308 |        NULL | Number of leaf pages in the index |
    | siemens_evse_cloud | position   | PRIMARY    | 2020-01-22 10:10:58 | size         |        353 |        NULL | Number of pages in the index      |
    +--------------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
    3 rows in set (0.00 sec)

直接看一下这个innodb_index_stats表中的各个列都是干嘛的：

字段名|	描述
----|----
database_name|	数据库名
table_name	|表名
index_name	|索引名
last_update	|本条记录最后更新时间
stat_name	|统计项的名称
stat_value	|对应的统计项的值
sample_size	|为生成统计数据而采样的页面数量
stat_description|	对应的统计项的描述

- n_leaf_pages：表示该索引的叶子节点占用多少页面。

- size：表示该索引共占用多少页面。

- n_diff_pfxNN：表示对应的索引列不重复的值有多少。其中的NN长得有点儿怪呀，啥意思呢？

其实NN可以被替换为01、02、03... 这样的数字。比如对于idx_key_part来说：

    n_diff_pfx01表示的是统计key_part1这单单一个列不重复的值有多少。
    
    n_diff_pfx02表示的是统计key_part1、key_part2这两个列组合起来不重复的值有多少。
    
    n_diff_pfx03表示的是统计key_part1、key_part2、key_part3这三个列组合起来不重复的值有多少。
    
    n_diff_pfx04表示的是统计key_part1、key_part2、key_part3、id这四个列组合起来不重复的值有多少。
    
    对于普通的二级索引，并不能保证它的索引列值是唯一的，比如对于idx_key1来说，key1列就可能有很多值重复的记录。
    此时只有在索引列上加上主键值才可以区分两条索引列值都一样的二级索引记录

### innodb_table_stats
        
    mysql> select * from mysql.innodb_table_stats where table_name like 'position' ;
    +--------------------+------------+---------------------+--------+----------------------+--------------------------+
    | database_name      | table_name | last_update         | n_rows | clustered_index_size | sum_of_other_index_sizes |
    +--------------------+------------+---------------------+--------+----------------------+--------------------------+
    | siemens_evse_cloud | position   | 2020-01-22 10:10:58 |  46631 |                  353 |                        0 |
    +--------------------+------------+---------------------+--------+----------------------+--------------------------+
    1 row in set (0.01 sec)
    
    
直接看一下这个innodb_table_stats表中的各个列都是干嘛的：

字段名	|描述
----|----
database_name|	数据库名
table_name	|表名
last_update	|本条记录最后更新时间
n_rows	|表中记录的条数, 数据是估计值
clustered_index_size|	表的聚簇索引占用的页面数量, 数据是估计值
sum_of_other_index_sizes|	表的其他索引占用的页面数量, 数据是估计值
    
    mysql> select count(*) from siemens_evse_cloud.position;
    +----------+
    | count(*) |
    +----------+
    |    46373 |
    +----------+
    1 row in set (0.08 sec)

#### n_rows 估计方法
按照一定算法（并不是纯粹随机的）选取几个叶子节点页面，计算每个页面中主键值记录数量，
然后计算平均一个页面中主键值的记录数量乘以全部叶子节点的数量就算是该表的n_rows值。

采样的页面数量设置，由下面可以看出默认为采样20页。
    
    mysql> show variables like 'innodb_stats_persistent_%';
    +--------------------------------------+-------+
    | Variable_name                        | Value |
    +--------------------------------------+-------+
    | innodb_stats_persistent_sample_pages | 20    |
    +--------------------------------------+-------+
    1 row in set (0.01 sec)

单独设置某一个表的采样页面数量
    
    CREATE TABLE 表名 (...) Engine=InnoDB, STATS_SAMPLE_PAGES = 具体的采样页面数量;
    ALTER TABLE 表名 Engine=InnoDB, STATS_SAMPLE_PAGES = 具体的采样页面数量;

### 定期更新统计数据
设置变量innodb_stats_auto_recalc

    mysql> show variables like 'innodb_stats_auto%';
    +--------------------------+-------+
    | Variable_name            | Value |
    +--------------------------+-------+
    | innodb_stats_auto_recalc | ON    |
    +--------------------------+-------+
    1 row in set (0.00 sec)

- 如果发生变动的记录数量超过了表大小的10%，并且自动重新计算统计数据的功能是打开的，那么服务器会重新进行一次统计数据的计算
- 自动重新计算统计数据的过程是异步发生的

单独设置某一个表的统计数据是否定期更新
    
    CREATE TABLE 表名 (...) Engine=InnoDB, STATS_AUTO_RECALC = (1|0);
    ALTER TABLE 表名 Engine=InnoDB, STATS_AUTO_RECALC = (1|0);

### 手动调用ANALYZE TABLE语句来更新统计信息
    mysql> analyze table siemens_evse_cloud.position;
    +-----------------------------+---------+----------+----------+
    | Table                       | Op      | Msg_type | Msg_text |
    +-----------------------------+---------+----------+----------+
    | siemens_evse_cloud.position | analyze | status   | OK       |
    +-----------------------------+---------+----------+----------+
    1 row in set (0.04 sec)
    mysql> select * from mysql.innodb_table_stats where table_name like 'position';
    +--------------------+------------+---------------------+--------+----------------------+--------------------------+
    | database_name      | table_name | last_update         | n_rows | clustered_index_size | sum_of_other_index_sizes |
    +--------------------+------------+---------------------+--------+----------------------+--------------------------+
    | siemens_evse_cloud | position   | 2020-04-17 10:34:02 |  45584 |                  353 |                        0 |
    +--------------------+------------+---------------------+--------+----------------------+--------------------------+
    1 row in set (0.00 sec)

可以看出n_rows更新了，但是好像离实际值更远了。

## 基于内存的非永久性统计数据
把系统变量innodb_stats_persistent的值设置为OFF时

非永久性的统计数据采样的页面数量是由innodb_stats_transient_sample_pages设置

    mysql> show variables like 'innodb_stats_transient_sample_pages';
    +-------------------------------------+-------+
    | Variable_name                       | Value |
    +-------------------------------------+-------+
    | innodb_stats_transient_sample_pages | 8     |
    +-------------------------------------+-------+
    1 row in set (0.00 sec)

## NULL值是否唯一
    mysql> select 1=NULL;
    +--------+
    | 1=NULL |
    +--------+
    |   NULL |
    +--------+
    1 row in set (0.00 sec)
    
    mysql> select 1 != NULL;
    +-----------+
    | 1 != NULL |
    +-----------+
    |      NULL |
    +-----------+
    1 row in set (0.00 sec)
    
    mysql> select NULL = NULL;
    +-------------+
    | NULL = NULL |
    +-------------+
    |        NULL |
    +-------------+
    1 row in set (0.00 sec)
    
    mysql> select NULL != NULL;
    +--------------+
    | NULL != NULL |
    +--------------+
    |         NULL |
    +--------------+

NULL是算不同呢，还是唯一呢，还是什么都不算呢有innodb_stats_method变量决定
    
    mysql> show variables like 'innodb_stats_method';
    +---------------------+-------------+
    | Variable_name       | Value       |
    +---------------------+-------------+
    | innodb_stats_method | nulls_equal |
    +---------------------+-------------+
    1 row in set (0.00 sec)

- nulls_equal：认为所有NULL值都是相等的。这个值也是innodb_stats_method的默认值。

如果某个索引列中NULL值特别多的话，这种统计方式会让优化器认为某个列中平均一个值重复次数特别多，所以倾向于不使用索引进行访问。

- nulls_unequal：认为所有NULL值都是不相等的。

如果某个索引列中NULL值特别多的话，这种统计方式会让优化器认为某个列中平均一个值重复次数特别少，所以倾向于使用索引进行访问。

- nulls_ignored：直接把NULL值忽略掉。

最好不在索引列中存放NULL值才是正解。