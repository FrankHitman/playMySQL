# cost optimize

## cost
对于InnoDB存储引擎来说，页是磁盘和内存之间交互的基本单位，设计MySQL的大叔规定读取一个页面花费的成本默认是1.0，
读取以及检测一条记录是否符合搜索条件的成本默认是0.2。1.0、0.2这些数字称之为成本常数

### 计算全表扫描的代价
    mysql> SHOW TABLE STATUS LIKE 'single_table'\G
    *************************** 1. row ***************************
               Name: single_table
             Engine: InnoDB
            Version: 10
         Row_format: Dynamic
               Rows: 9693
     Avg_row_length: 163
        Data_length: 1589248
    Max_data_length: 0
       Index_length: 2752512
          Data_free: 4194304
     Auto_increment: 10001
        Create_time: 2018-12-10 13:37:23
        Update_time: 2018-12-10 13:38:03
         Check_time: NULL
          Collation: utf8_general_ci
           Checksum: NULL
     Create_options:
            Comment:
    1 row in set (0.01 sec)
    
- Rows

本选项表示表中的记录条数。对于使用MyISAM存储引擎的表来说，该值是准确的，对于使用InnoDB存储引擎的表来说，该值是一个估计值。
从查询结果我们也可以看出来，由于我们的single_table表是使用InnoDB存储引擎的，所以虽然实际上表中有10000条记录，但是SHOW TABLE STATUS显示的Rows值只有9693条记录。

- Data_length

本选项表示表占用的存储空间字节数。使用MyISAM存储引擎的表来说，该值就是数据文件的大小，
对于使用InnoDB存储引擎的表来说，该值就相当于聚簇索引占用的存储空间大小，也就是说可以这样计算该值的大小：

    Data_length = 聚簇索引的页面数量 x 每个页面的大小
我们的single_table使用默认16KB的页面大小，而上边查询结果显示Data_length的值是1589248，所以我们可以反向来推导出聚簇索引的页面数量：

    聚簇索引的页面数量 = 1589248 ÷ 16 ÷ 1024 = 97

- I/O成本

```
97 x 1.0 + 1.1 = 98.1
97指的是聚簇索引占用的页面数，1.0指的是加载一个页面的成本常数，后边的1.1是一个微调值。
```
- CPU成本：
```
9693 x 0.2 + 1.0 = 1939.6
9693指的是统计数据中表的记录数，对于InnoDB存储引擎来说是一个估计值，0.2指的是访问一条记录所需的成本常数，后边的1.0是一个微调值，我们不用在意。
```

- 总成本：
```
98.1 + 1939.6 = 2037.7
```

### 索引查詢的成本
#### index dive

    SELECT * FROM single_table WHERE key1 IN ('aa1', 'aa2', 'aa3', ... , 'zzz');
    
这种通过直接访问索引对应的B+树来计算某个范围区间对应的索引记录条数的方式称之为index dive。

    mysql> SHOW VARIABLES LIKE '%dive%';
    +---------------------------+-------+
    | Variable_name             | Value |
    +---------------------------+-------+
    | eq_range_index_dive_limit | 200   |
    +---------------------------+-------+
    1 row in set (0.04 sec)

如果我们的IN语句中的参数个数小于200个的话，将使用index dive的方式计算各个单点区间对应的记录条数，
如果大于或等于200个的话，可就不能使用index dive了，要使用所谓的索引统计数据来进行估算
#### 索引统计数据
    Database changed
    mysql> show index from charger;
    +---------+------------+------------------------------------------------------------------+--------------+----------------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
    | Table   | Non_unique | Key_name                                                         | Seq_in_index | Column_name                | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
    +---------+------------+------------------------------------------------------------------+--------------+----------------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
    | charger |          0 | PRIMARY                                                          |            1 | id                         | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
    | charger |          1 | production_line_charger_2d826a2df12b3cbe678ee33366d0f6220e82858d |            1 | production_line_charger_id | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
    | charger |          1 | idx_created_at                                                   |            1 | created_at                 | A         |           0 |     NULL | NULL   | YES  | BTREE      |         |               |
    | charger |          1 | idx_fk_traceability_num                                          |            1 | fk_traceability_num        | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
    +---------+------------+------------------------------------------------------------------+--------------+----------------------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
    4 rows in set (0.01 sec)
    mysql> SHOW INDEX FROM single_table;
    +--------------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
    | Table        | Non_unique | Key_name     | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
    +--------------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
    | single_table |          0 | PRIMARY      |            1 | id          | A         |       9693  |     NULL | NULL   |      | BTREE      |         |               |
    | single_table |          0 | idx_key2     |            1 | key2        | A         |       9693  |     NULL | NULL   | YES  | BTREE      |         |               |
    | single_table |          1 | idx_key1     |            1 | key1        | A         |        968 |     NULL | NULL   | YES  | BTREE      |         |               |
    | single_table |          1 | idx_key3     |            1 | key3        | A         |        799 |     NULL | NULL   | YES  | BTREE      |         |               |
    | single_table |          1 | idx_key_part |            1 | key_part1   | A         |        9673 |     NULL | NULL   | YES  | BTREE      |         |               |
    | single_table |          1 | idx_key_part |            2 | key_part2   | A         |        9999 |     NULL | NULL   | YES  | BTREE      |         |               |
    | single_table |          1 | idx_key_part |            3 | key_part3   | A         |       10000 |     NULL | NULL   | YES  | BTREE      |         |               |
    +--------------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
    7 rows in set (0.01 sec)

字段意義

属性名|	描述
----|----
Table|	索引所属表的名称。
Non_unique	|索引列的值是否是唯一的，聚簇索引和唯一二级索引的该列值为0，普通二级索引该列值为1。
Key_name	|索引的名称。
Seq_in_index|	索引列在索引中的位置，从1开始计数。比如对于联合索引idx_key_part，来说，key_part1、key_part2和key_part3对应的位置分别是1、2、3。
Column_name	|索引列的名称。
Collation	|索引列中的值是按照何种排序方式存放的，值为A时代表升序存放，为NULL时代表降序存放。
Cardinality	|索引列中不重复值的数量。后边我们会重点看这个属性的。
Sub_part	|对于存储字符串或者字节串的列来说，有时候我们只想对这些串的前n个字符或字节建立索引，这个属性表示的就是那个n值。如果对完整的列建立索引的话，该属性的值就是NULL。
Packed	|索引列如何被压缩，NULL值表示未被压缩。这个属性我们暂时不了解，可以先忽略掉。
Null	|该索引列是否允许存储NULL值。
Index_type|	使用索引的类型，我们最常见的就是BTREE，其实也就是B+树索引。
Comment|	索引列注释信息。
Index_comment|	索引注释信息。

Cardinality直译过来就是基数的意思，表示索引列中不重复值的个数。
比如对于一个一万行记录的表来说，某个索引列的Cardinality属性是10000，那意味着该列中没有重复的值，
如果Cardinality属性是1的话，就意味着该列的值全部是重复的。

对于InnoDB存储引擎来说，使用SHOW INDEX语句展示出来的某个索引列的Cardinality属性是一个估计值，并不是精确的。

当IN语句中的参数个数大于或等于系统变量eq_range_index_dive_limit的值的话，就不会使用index dive的方式计算各个单点区间对应的索引记录条数，
而是使用索引统计数据，这里所指的索引统计数据指的是这两个值：

- 使用SHOW TABLE STATUS展示出的Rows值，也就是一个表中有多少条记录。
- 使用SHOW INDEX语句展示出的Cardinality属性。

结合上一个Rows统计数据，我们可以针对索引列，计算出平均一个值重复多少次。

    一个值的重复次数 ≈ Rows ÷ Cardinality

```SELECT * FROM single_table WHERE key1 IN ('aa1', 'aa2', 'aa3', ... , 'zzz');```
设IN语句中有20000个参数的话，就直接使用统计数据来估算这些参数需要单点区间对应的记录条数了，每个参数大约对应10条记录，
所以总共需要回表的记录数就是：

    20000 x 10 = 200000

### 连接查询的成本
- 单次查询驱动表的成本
- 多次查询被驱动表的成本（具体查询多少次取决于对驱动表查询的结果集中有多少条记录）

我们把对驱动表进行查询后得到的记录条数称之为驱动表的扇出（英文名：fanout）

- 如果使用的是全表扫描的方式执行的单表查询，那么计算驱动表扇出时需要猜满足搜索条件的记录到底有多少条。
- 如果使用的是索引执行的单表扫描，那么计算驱动表扇出的时候需要猜满足除使用到对应索引的搜索条件外的其他搜索条件的记录有多少条。

设计MySQL的大叔把这个猜的过程称之为condition filtering。

连接查询的成本计算公式是这样的：
```
连接查询总成本 = 单次访问驱动表的成本 + 驱动表扇出数 x 单次访问被驱动表的成本
SELECT * FROM single_table AS s1 INNER JOIN single_table2 AS s2 
    ON s1.key1 = s2.common_field 
    WHERE s1.key2 > 10 AND s1.key2 < 1000 AND 
          s2.key2 > 1000 AND s2.key2 < 2000;

使用s1作为驱动表的情况
使用idx_key2访问s1的成本 + s1的扇出 × 使用idx_key2访问s2的成本

使用s2作为驱动表的情况
使用idx_key2访问s2的成本 + s2的扇出 × 使用idx_key1访问s1的成本

```

### 多表连接的成本分析
对于n表连接的话，则有 n × (n-1) × (n-2) × ··· × 1种连接顺序，就是n的阶乘种连接顺序，也就是n!
- 提前结束某种顺序的成本评估
- 系统变量optimizer_search_depth
- 根据某些规则压根儿就不考虑某些连接顺序

## 调节成本常数  
mysql.server_cost
 
    mysql> SHOW TABLES FROM mysql LIKE '%cost%';
    +--------------------------+
    | Tables_in_mysql (%cost%) |
    +--------------------------+
    | engine_cost              |
    | server_cost              |
    +--------------------------+
    2 rows in set (0.01 sec)
    
    mysql> select * from mysql.server_cost;
    +------------------------------+------------+---------------------+---------+
    | cost_name                    | cost_value | last_update         | comment |
    +------------------------------+------------+---------------------+---------+
    | disk_temptable_create_cost   |       NULL | 2020-01-22 04:17:01 | NULL    |
    | disk_temptable_row_cost      |       NULL | 2020-01-22 04:17:01 | NULL    |
    | key_compare_cost             |       NULL | 2020-01-22 04:17:01 | NULL    |
    | memory_temptable_create_cost |       NULL | 2020-01-22 04:17:01 | NULL    |
    | memory_temptable_row_cost    |       NULL | 2020-01-22 04:17:01 | NULL    |
    | row_evaluate_cost            |       NULL | 2020-01-22 04:17:01 | NULL    |
    +------------------------------+------------+---------------------+---------+
    6 rows in set (0.00 sec)

成本常数名称|	默认值|	描述
----|----|----
disk_temptable_create_cost	|40.0|	创建基于磁盘的临时表的成本，如果增大这个值的话会让优化器尽量少的创建基于磁盘的临时表。
disk_temptable_row_cost	|1.0|	向基于磁盘的临时表写入或读取一条记录的成本，如果增大这个值的话会让优化器尽量少的创建基于磁盘的临时表。
key_compare_cost	|0.1|	两条记录做比较操作的成本，多用在排序操作上，如果增大这个值的话会提升filesort的成本，让优化器可能更倾向于使用索引完成排序而不是filesort。
memory_temptable_create_cost	|2.0|	创建基于内存的临时表的成本，如果增大这个值的话会让优化器尽量少的创建基于内存的临时表。
memory_temptable_row_cost	|0.2|	向基于内存的临时表写入或读取一条记录的成本，如果增大这个值的话会让优化器尽量少的创建基于内存的临时表。
row_evaluate_cost	|0.2|	这个就是我们之前一直使用的检测一条记录是否符合搜索条件的成本，增大这个值可能让优化器更倾向于使用索引而不是直接全表扫描。

修改成本常數
```
UPDATE mysql.server_cost 
    SET cost_value = 0.4
    WHERE cost_name = 'row_evaluate_cost';
FLUSH OPTIMIZER_COSTS;
```

mysql.engine_cost

    mysql> select * from mysql.engine_cost;
    +-------------+-------------+------------------------+------------+---------------------+---------+
    | engine_name | device_type | cost_name              | cost_value | last_update         | comment |
    +-------------+-------------+------------------------+------------+---------------------+---------+
    | default     |           0 | io_block_read_cost     |       NULL | 2020-01-22 04:17:01 | NULL    |
    | default     |           0 | memory_block_read_cost |       NULL | 2020-01-22 04:17:01 | NULL    |
    +-------------+-------------+------------------------+------------+---------------------+---------+
    2 rows in set (0.01 sec)

成本常数名称	|默认值|	描述
----|----|----
io_block_read_cost	|1.0|	从磁盘上读取一个块对应的成本。请注意我使用的是块，而不是页这个词儿。对于InnoDB存储引擎来说，一个页就是一个块，不过对于MyISAM存储引擎来说，默认是以4096字节作为一个块的。增大这个值会加重I/O成本，可能让优化器更倾向于选择使用索引执行查询而不是执行全表扫描。
memory_block_read_cost	|1.0|	与上一个参数类似，只不过衡量的是从内存中读取一个块对应的成本。

為什麼都是1.0？这主要是因为在MySQL目前的实现中，并不能准确预测某个查询需要访问的块中有哪些块已经加载到内存中，有哪些块还停留在磁盘上，
所以设计MySQL的大叔们很粗暴的认为不管这个块有没有加载到内存中，使用的成本都是1.0




