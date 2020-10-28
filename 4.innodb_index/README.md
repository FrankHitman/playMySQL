# Index

## 索引的代價
- 空間代價
- 時間代價

每次對表中數據進行增、刪、改操作時，都需要去修改各個B+樹索引，會對節點和紀錄的排序造成破壞，
所以存儲引擎需要額外時間進行一些紀錄移位，頁面分裂、頁面回收等操作來維護紀錄的排序。

## 索引用于查询
### 匹配列前綴
建立如下聯合索引

    CREATE TABLE person_info(
        id INT NOT NULL auto_increment,
        name VARCHAR(100) NOT NULL,
        birthday DATE NOT NULL,
        phone_number CHAR(11) NOT NULL,
        country varchar(100) NOT NULL,
        PRIMARY KEY (id),
        KEY idx_name_birthday_phone_number (name, birthday, phone_number)
    );

对于字符串类型的索引列来说，我们只匹配它的前缀也是可以快速定位记录的，
比方说我们想查询名字以'As'开头的记录，那就可以这么写查询语句：

    SELECT * FROM person_info WHERE name LIKE 'As%';
    
 但是如果只给出后缀或者中间的某个字符串，那麼只能全表掃描而無法使用索引

    SELECT * FROM person_info WHERE name LIKE '%As%';
  
### 匹配范围值
不过在使用联合进行范围查找的时候需要注意，如果对多个列同时进行范围查找的话，
只有对索引最左边的那个列进行范围查找的时候才能用到B+树索引，比方说这样：
   
    SELECT * FROM person_info WHERE name > 'Asa' AND name < 'Barlow' AND birthday > '1980-01-01';

- 通过条件name > 'Asa' AND name < 'Barlow'来对name进行范围，查找的结果可能有多条name值不同的记录，
- 对这些name值不同的记录继续通过birthday > '1980-01-01'条件继续过滤。這一步需要在查詢的紀錄中進行逐一比對，用不到聯合索引中的birthday

### 精确匹配某一列并范围匹配另外一列

    SELECT * FROM person_info WHERE name = 'Ashburn' AND birthday > '1980-01-01' AND birthday < '2000-12-31' AND phone_number > '15100000000';

- name = 'Ashburn'，对name列进行精确查找，当然可以使用B+树索引了。
- birthday > '1980-01-01' AND birthday < '2000-12-31'，由于name列是精确查找，所以通过name = 'Ashburn'条件查找后得到的结果的name值都是相同的，它们会再按照birthday的值进行排序。所以此时对birthday列进行范围查找是可以用到B+树索引的。
- phone_number > '15100000000'，通过birthday的范围查找的记录的birthday的值可能不同，所以这个条件无法再利用B+树索引了，只能遍历上一步查询得到的记录。    
    
## 索引用于排序
### 文件排序(filesort)    
我们在写查询语句的时候经常需要对查询出来的记录通过ORDER BY子句按照某种规则进行排序。
一般情况下，我们只能把记录都加载到内存中，再用一些排序算法，比如快速排序、归并排序、吧啦吧啦排序等等在内存中对这些记录进行排序，
有的时候可能查询的结果集太大以至于不能在内存中进行排序的话，还可能暂时借助磁盘的空间来存放中间结果，排序操作完成后再把排好序的结果集返回到客户端。
在MySQL中，把这种在内存中或者磁盘上进行排序的方式统称为文件排序（英文名：filesort），相對於使用索引是比較慢的排序。

### order by index desc 可以使用索引嗎？
[mysql8.0支持倒序索引](https://dev.mysql.com/doc/refman/8.0/en/descending-indexes.html)
但是mysql5.7也是可以使用默認的升序索引為倒序使用的。

定義如下表

    create table siemens_evse_cloud.charger
    (
        id                         varchar(36)                        not null
            primary key,
        production_line_charger_id varchar(36)                        not null,
        name                       varchar(32)                        null,
        created_at                 timestamp                          null,
        fk_traceability_num        varchar(32)                        null,
        constraint production_line_charger_2d826a2df12b3cbe678ee33366d0f6220e82858d
            foreign key (production_line_charger_id) references siemens_evse_cloud.production_line_charger (id)
    );
    
    create index idx_created_at
        on siemens_evse_cloud.charger (created_at);
    
    create index idx_fk_traceability_num
        on siemens_evse_cloud.charger (fk_traceability_num);

使用explain語句來查看查詢優化情況：

    mysql> explain select * from charger order by created_at;
    +------+-------------+---------+------+---------------+------+---------+------+------+----------------+
    | id   | select_type | table   | type | possible_keys | key  | key_len | ref  | rows | Extra          |
    +------+-------------+---------+------+---------------+------+---------+------+------+----------------+
    |    1 | SIMPLE      | charger | ALL  | NULL          | NULL | NULL    | NULL | 1    | Using filesort |
    +------+-------------+---------+------+---------------+------+---------+------+------+----------------+
    1 row in set (0.01 sec)
    mysql> explain select * from charger order by created_at desc;
    +------+-------------+---------+------+---------------+------+---------+------+------+----------------+
    | id   | select_type | table   | type | possible_keys | key  | key_len | ref  | rows | Extra          |
    +------+-------------+---------+------+---------------+------+---------+------+------+----------------+
    |    1 | SIMPLE      | charger | ALL  | NULL          | NULL | NULL    | NULL | 1    | Using filesort |
    +------+-------------+---------+------+---------------+------+---------+------+------+----------------+
    1 row in set (0.00 sec)
    


從上面可以看出它使用的是文件排序，因為select * 獲取所有的列，而不僅僅是索引列，就算它通過二級索引獲取到數據列，它還是需要去聚簇索引中回表獲取所有列，
所以查询优化器觉得还是直接全表扫描比使用二级索引更快，因此是文件排序。
參考[mysql5.7手冊](https://dev.mysql.com/doc/refman/5.7/en/order-by-optimization.html#order-by-index-use)

如果帶limit呢？

    mysql> explain select * from charger order by created_at limit 10;
    +------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
    | id   | select_type | table   | type  | possible_keys | key            | key_len | ref  | rows | Extra |
    +------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
    |    1 | SIMPLE      | charger | index | NULL          | idx_created_at | 8       | NULL | 1    |       |
    +------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
    1 row in set (0.00 sec)
    mysql> explain select * from charger order by created_at desc limit 10;
    +------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
    | id   | select_type | table   | type  | possible_keys | key            | key_len | ref  | rows | Extra |
    +------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
    |    1 | SIMPLE      | charger | index | NULL          | idx_created_at | 8       | NULL | 1    |       |
    +------+-------------+---------+-------+---------------+----------------+---------+------+------+-------+
    1 row in set (0.00 sec)

從上面可以看出，查詢優化器使用了索引排序。
綜上所述，mysql5.7中的索引在倒序排序並且有limit查询時候是起到作用的。

### 不可以使用索引进行排序的几种情况
- ASC、DESC混用

```SELECT * FROM person_info ORDER BY name, birthday DESC LIMIT 10;```
    
- WHERE子句中出现非排序使用到的索引列 

```SELECT * FROM person_info WHERE country = 'China' ORDER BY name LIMIT 10;```
    
- 排序列包含非同一个索引的列    

```SELECT * FROM person_info ORDER BY name, country LIMIT 10;```

- 排序列使用了复杂的表达式

```SELECT * FROM person_info ORDER BY UPPER(name) LIMIT 10;```


## 索引用于分组，前提是分组顺序与索引顺序一致
```SELECT name, birthday, phone_number, COUNT(*) FROM person_info GROUP BY name, birthday, phone_number```
- 先把记录按照name值进行分组，所有name值相同的记录划分为一组。
- 将每个name值相同的分组里的记录再按照birthday的值进行分组，将birthday值相同的记录放到一个小分组里，所以看起来就像在一个大分组里又化分了好多小分组。
- 再将上一步中产生的小分组按照phone_number的值分成更小的分组，所以整体上看起来就像是先把记录分成一个大分组，然后把大分组分成若干个小分组，然后把若干个小分组再细分成更多的小小分组。

## 查询优化
### 回表
特点
- 会使用到两个B+树索引，一个二级索引，一个聚簇索引。
- 访问二级索引使用顺序I/O，访问聚簇索引使用随机I/O。

需要回表的记录越多，使用二级索引的性能就越低，甚至让某些查询宁愿使用全表扫描也不使用二级索引

### 查询优化器
查询优化器会事先对表中的记录计算一些统计数据，然后再利用这些统计数据根据查询的条件来计算一下需要回表的记录数，
需要回表的记录数越多，就越倾向于使用全表扫描，反之倾向于使用二级索引 + 回表的方式。
当然优化器做的分析工作不仅仅是这么简单，但是大致上是个这个过程。
一般情况下，限制查询获取较少的记录数会让优化器更倾向于选择使用二级索引 + 回表的方式进行查询，因为回表的记录越少，性能提升就越高

### 覆盖索引
为了彻底告别回表操作带来的性能损耗，我们建议：最好在查询列表里只包含索引列

    SELECT name, birthday, phone_number FROM person_info WHERE name > 'Asa' AND name < 'Barlow'

    SELECT name, birthday, phone_number  FROM person_info ORDER BY name, birthday, phone_number;

## 如何挑选索引列

### 只为用于搜索、排序或分组的列创建索引

### 考虑列的基数，最好为那些列的基数大的列建立索引

列的基数指的是某一列中不重复数据的个数，比方说某个列包含值2, 5, 8, 2, 5, 8, 2, 5, 8，虽然有9条记录，但该列的基数却是3。
也就是说，在记录行数一定的情况下，列的基数越大，该列中的值越分散，列的基数越小，该列中的值越集中。

### 索引列的类型尽量小

我们在定义表结构的时候要显式的指定列的类型，以整数类型为例，有TINYINT、MEDIUMINT、INT、BIGINT这么几种，它们占用的存储空间依次递增，
我们这里所说的类型大小指的就是该类型表示的数据范围的大小。能表示的整数范围当然也是依次递增，
如果我们想要对某个整数列建立索引的话，在表示的整数范围允许的情况下，尽量让索引列使用较小的类型，
比如我们能使用INT就不要使用BIGINT，能使用MEDIUMINT就不要使用INT～ 这是因为：

- 数据类型越小，在查询时进行的比较操作越快（这是CPU层次的东东）
- 数据类型越小，索引占用的存储空间就越少，在一个数据页内就可以放下更多的记录，从而减少磁盘I/O带来的性能损耗，
也就意味着可以把更多的数据页缓存在内存中，从而加快读写效率。

这个建议对于表的主键来说更加适用，因为不仅是聚簇索引中会存储主键值，其他所有的二级索引的节点处都会存储一份记录的主键值，
如果主键适用更小的数据类型，也就意味着节省更多的存储空间和更高效的I/O。

### 索引字符串值的前缀

只对字符串的前几个字符进行索引也就是说在二级索引的记录中只保留字符串前几个字符。这样在查找记录时虽然不能精确的定位到记录的位置，
但是能定位到相应前缀所在的位置，然后根据前缀相同的记录的主键值回表查询完整的字符串值，再对比就好了。
这样只在B+树中存储字符串的前几个字符的编码，既节约空间，又减少了字符串的比较时间，还大概能解决排序的问题
    
    CREATE TABLE person_info(
        name VARCHAR(100) NOT NULL,
        birthday DATE NOT NULL,
        phone_number CHAR(11) NOT NULL,
        country varchar(100) NOT NULL,
        KEY idx_name_birthday_phone_number (name(10), birthday, phone_number)
    ); 

索引列前缀对排序的影响

    SELECT * FROM person_info ORDER BY name LIMIT 10;
    
因为二级索引中不包含完整的name列信息，所以无法对前十个字符相同，后边的字符不同的记录进行排序，所以将会使用文件排序    
    
### 让索引列在比较表达式中单独出现
1. WHERE my_col * 2 < 4
2. WHERE my_col < 4/2

2可以使用到索引，2的查询效率高。

如果索引列在比较表达式中不是以单独列的形式出现，而是以某个表达式，或者函数调用形式出现的话，是用不到索引的

### 主键插入顺序
最好让插入的记录的主键值依次递增，这样就不会发生这样的性能损耗了（例如页面分裂和紀錄移位）。
所以我们建议：让主键具有AUTO_INCREMENT，让存储引擎自己为表生成主键，而不是我们手动插入。

但是自增加主鍵在分布式系統中分庫會導致主鍵重複的問題，還有restful風格API中id洩漏問題，所以根據生產環境選擇uuid。

### 定位并删除表中的重复和冗余索引

    