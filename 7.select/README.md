# select

## 訪問方法
MySQL执行查询语句的方式称之为访问方法或者访问类型

- 使用全表扫描进行查询
- 使用索引进行查询
    - 针对主键或唯一二级索引的等值查询
    - 针对普通二级索引的等值查询
    - 针对索引列的范围查询
    - 直接扫描整个索引


    CREATE TABLE single_table (
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

## const
    SELECT * FROM single_table WHERE id = 1438;
    SELECT * FROM single_table WHERE key2 = 3841;
這些走聚簇索引或者二級唯一索引，比較快

## ref
通过索引列进行等值比较后可能匹配到多条连续的记录，而不是像主键或者唯一二级索引那样最多只能匹配1条记录，
所以这种ref访问方法比const差了那么一丢丢，但是在二级索引等值比较时匹配的记录数较少时的效率还是很高的
- 普通二級索引


    SELECT * FROM single_table WHERE key1 = 'abc';

由于普通二级索引并不限制索引列值的唯一性，所以可能找到多条对应的记录，也就是说使用二级索引来执行查询的代价取决于等值匹配到的二级索引记录条数。
如果匹配的记录较少，则回表的代价还是比较低的，所以MySQL可能选择使用索引而不是全表扫描的方式来执行查询。

- 二級索引為null的情況


    SELECT * FROM single_table WHERE key2 IS NULL;
- 聯合索引於常數等值比較


    SELECT * FROM single_table WHERE key_part1 = 'god like';
    SELECT * FROM single_table WHERE key_part1 = 'god like' AND key_part2 = 'legendary';
    SELECT * FROM single_table WHERE key_part1 = 'god like' AND key_part2 = 'legendary' AND key_part3 = 'penta kill';

以下除外，因為key_part2不是常量等值比較

    SELECT * FROM single_table WHERE key_part1 = 'god like' AND key_part2 > 'legendary';
    
## ref_or_null
有时候我们不仅想找出某个二级索引列的值等于某个常数的记录，还想把该列的值为NULL的记录也找出来，
当使用二级索引而不是全表扫描的方式执行该查询时，这种类型的查询使用的访问方法就称为ref_or_null
    
    SELECT * FROM single_table WHERE key1 = 'abc' OR key1 IS NULL;

## range
利用索引进行范围匹配的访问方法称之为：range。

    SELECT * FROM single_table WHERE key2 IN (1438, 6328) OR (key2 >= 38 AND key2 <= 79);

## index
采用遍历二级索引记录而不需要去聚簇索引中回表的执行方式称之为：index

    SELECT key_part1, key_part2, key_part3 FROM single_table WHERE key_part2 = 'abc';

## all
使用全表扫描聚簇索引的执行查询的方式称之为：all

## 注意事項
### 二级索引 + 回表
    SELECT * FROM single_table WHERE key1 = 'abc' AND key2 > 1000;

- 步骤1：使用二级索引定位记录的阶段，也就是根据条件key1 = 'abc'从idx_key1索引代表的B+树中找到对应的二级索引记录。

- 步骤2：回表阶段，也就是根据上一步骤中找到的记录的主键值进行回表操作，也就是到聚簇索引中找到对应的完整的用户记录，再根据条件key2 > 1000到完整的用户记录继续过滤。将最终符合过滤条件的记录返回给用户。

### 明确range访问方法使用的范围区间

其实对于B+树索引来说，只要索引列和常数使用=、<=>、IN、NOT IN、IS NULL、IS NOT NULL、>、<、>=、<=、BETWEEN、!=（不等于也可以写成<>）
或者LIKE操作符（只前綴匹配）连接起来，就可以产生一个所谓的区间。

### 有的搜索条件无法使用索引的情况
    SELECT * FROM single_table WHERE key2 > 100 AND common_field = 'abc';
    化簡為：
    SELECT * FROM single_table WHERE key2 > 100 AND TRUE;
    也就是：
    SELECT * FROM single_table WHERE key2 > 100;
 
 之所以把用不到索引的搜索条件替换为TRUE，是因为我们不打算使用这些条件进行在该索引上进行过滤，
 所以不管索引的记录满不满足这些条件，我们都把它们选取出来，待到之后回表的时候再使用它们过滤。

另外一種OR情況

    SELECT * FROM single_table WHERE key2 > 100 OR common_field = 'abc';
    化簡為：
    SELECT * FROM single_table WHERE key2 > 100 OR TRUE;
    也就是：
    SELECT * FROM single_table WHERE TRUE;
 额，这也就说说明如果我们强制使用idx_key2执行查询的话，对应的范围区间就是(-∞, +∞)，
 也就是需要将全部二级索引的记录进行回表，这个代价肯定比直接全表扫描都大了。
 也就是说一个使用到索引的搜索条件和没有使用该索引的搜索条件使用OR连接起来后是无法使用该索引的。
 
### 复杂搜索条件下找出范围匹配的区间
    SELECT * FROM single_table WHERE 
            (key1 > 'xyz' AND key2 = 748 ) OR
            (key1 < 'abc' AND key1 > 'lmn') OR
            (key1 LIKE '%suf' AND key1 > 'zzz' AND (key2 < 8000 OR common_field = 'abc')) ;

#### 假设我们使用idx_key1执行查询進行化簡

以key1二級索引查詢，那麼有关key2、common_field列，key1 LIKE '%suf'使用不到索引，所以把这些搜索条件替换为TRUE
           
    SELECT * FROM single_table WHERE 
            (key1 > 'xyz' AND TRUE ) OR
            (key1 < 'abc' AND key1 > 'lmn') OR
            (TRUE AND key1 > 'zzz' AND (key2 < 8000 OR TRUE)) ;

    SELECT * FROM single_table WHERE 
            (key1 > 'xyz') OR
            (key1 < 'abc' AND key1 > 'lmn') OR
            (key1 > 'zzz' AND TRUE) ;
    
    SELECT * FROM single_table WHERE 
            (key1 > 'xyz') OR
            (key1 < 'abc' AND key1 > 'lmn') OR
            (key1 > 'zzz') ;

簡化永遠為TRUE或者FALSE的條件

    SELECT * FROM single_table WHERE 
            (key1 > 'xyz') OR
            FALSE OR
            (key1 > 'zzz') ;

    SELECT * FROM single_table WHERE 
            (key1 > 'xyz') OR (key1 > 'zzz') ;
    
    SELECT * FROM single_table WHERE 
            (key1 > 'xyz') ;
                   
 
#### 假设我们使用idx_key2执行查询
    SELECT * FROM single_table WHERE 
            (TRUE AND key2 = 748 ) OR
            (TRUE AND TRUE) OR
            (TRUE AND TRUE AND (key2 < 8000 OR TRUE)) ;

    SELECT * FROM single_table WHERE 
            (key2 = 748 ) OR
            ((TRUE)) ;

    SELECT * FROM single_table WHERE TRUE;
    
如果以key2索引查詢需要掃描全部二級索引，然後再去回表，不如直接全表查詢了

### 索引合併
#### Intersection合并，交集

    SELECT * FROM single_table WHERE key1 = 'a' AND key3 = 'b';

虽然读取多个二级索引比读取一个二级索引消耗性能，但是读取二级索引的操作是顺序I/O，而回表操作是随机I/O，
所以如果只读取一个二级索引时需要回表的记录数特别多，而读取多个二级索引之后取交集的记录数非常少，
当节省的因为回表而造成的性能损耗比访问多个二级索引带来的性能损耗更高时，读取多个二级索引后取交集比只读取一个二级索引的成本更低。

- 情况一：二级索引列是等值匹配的情况，对于联合索引来说，在联合索引中的每个列都必须等值匹配，不能出现只匹配部分列的情况


    SELECT * FROM single_table WHERE key1 = 'a' AND key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c';
除外

    SELECT * FROM single_table WHERE key1 > 'a' AND key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c';
    SELECT * FROM single_table WHERE key1 = 'a' AND key_part1 = 'a';
    
 - 情况二：主键列可以是范围匹配
    
    
    SELECT * FROM single_table WHERE id > 100 AND key1 = 'a';
 别忘了二级索引的记录中都带有主键值的，所以可以在从idx_key1中获取到的主键值上直接运用条件id > 100过滤就行了，这样多简单。
 所以涉及主键的搜索条件只不过是为了从别的二级索引得到的结果集中过滤记录罢了，是不是等值匹配不重要。
 
 也就是说即使情况一、情况二成立，也不一定发生Intersection索引合并，
 
 #### Union合并，並集
    SELECT * FROM single_table WHERE key1 = 'a' OR key3 = 'b'
 - 情况一：二级索引列是等值匹配的情况，对于联合索引来说，在联合索引中的每个列都必须等值匹配，不能出现只出现匹配部分列的情况。
 
 
    SELECT * FROM single_table WHERE key1 = 'a' OR ( key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c');
 而下边这两个查询就不能进行Union索引合并：
     
     SELECT * FROM single_table WHERE key1 > 'a' OR (key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c');
     SELECT * FROM single_table WHERE key1 = 'a' OR key_part1 = 'a';
 第一个查询是因为对key1进行了范围匹配，第二个查询是因为联合索引idx_key_part中的key_part2和key_part3列并没有出现在搜索条件中，所以这两个查询不能进行Union索引合并。
 
 - 情况二：主键列可以是范围匹配
 
 - 情况三：使用Intersection索引合并的搜索条件
 
 这种情况其实也挺好理解，就是搜索条件的某些部分使用Intersection索引合并的方式得到的主键集合和其他方式得到的主键集合取交集，比方说这个查询：
 
    SELECT * FROM single_table WHERE key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c' OR (key1 = 'a' AND key3 = 'b');
 优化器可能采用这样的方式来执行这个查询：
 
    - 先按照搜索条件key1 = 'a' AND key3 = 'b'从索引idx_key1和idx_key3中使用Intersection索引合并的方式得到一个主键集合。
    
    - 再按照搜索条件key_part1 = 'a' AND key_part2 = 'b' AND key_part3 = 'c'从联合索引idx_key_part中得到另一个主键集合。
    
    - 采用Union索引合并的方式把上述两个主键集合取并集，然后进行回表操作，将结果返回给用户。

#### Sort-Union合并 
    SELECT * FROM single_table WHERE key1 < 'a' OR key3 > 'z'
 
 - 先根据key1 < 'a'条件从idx_key1二级索引中获取记录，并按照记录的主键值进行排序
 
 - 再根据key3 > 'z'条件从idx_key3二级索引中获取记录，并按照记录的主键值进行排序
 
 - 因为上述的两个二级索引主键值都是排好序的，剩下的操作和Union索引合并方式就一样了。
 
 这种Sort-Union索引合并比单纯的Union索引合并多了一步对二级索引记录的主键值排序的过程。