
### MySQL中的utf8和utf8mb4

-   utf8mb3：阉割过的utf8字符集，只使用1～3个字节表示字符。

-   utf8mb4：正宗的utf8字符集，使用1～4个字节表示字符。

有一点需要大家十分的注意，在MySQL中utf8是utf8mb3的别名，所以之后在MySQL中提到utf8就意味着使用1~3个字节来表示一个字符，如果大家有使用4字节编码一个字符的情况，比如存储一些emoji表情啥的，那请使用utf8mb4。

### 字符集的查看

```show charset;```
```
MariaDB [(none)]> show character set;
+----------+-----------------------------+---------------------+--------+
| Charset  | Description                 | Default collation   | Maxlen |
+----------+-----------------------------+---------------------+--------+
| big5     | Big5 Traditional Chinese    | big5_chinese_ci     |      2 |
| dec8     | DEC West European           | dec8_swedish_ci     |      1 |
| cp850    | DOS West European           | cp850_general_ci    |      1 |
| hp8      | HP West European            | hp8_english_ci      |      1 |
| koi8r    | KOI8-R Relcom Russian       | koi8r_general_ci    |      1 |
| latin1   | cp1252 West European        | latin1_swedish_ci   |      1 |
| latin2   | ISO 8859-2 Central European | latin2_general_ci   |      1 |
| swe7     | 7bit Swedish                | swe7_swedish_ci     |      1 |
| ascii    | US ASCII                    | ascii_general_ci    |      1 |
| ujis     | EUC-JP Japanese             | ujis_japanese_ci    |      3 |
| sjis     | Shift-JIS Japanese          | sjis_japanese_ci    |      2 |
| hebrew   | ISO 8859-8 Hebrew           | hebrew_general_ci   |      1 |
| tis620   | TIS620 Thai                 | tis620_thai_ci      |      1 |
| euckr    | EUC-KR Korean               | euckr_korean_ci     |      2 |
| koi8u    | KOI8-U Ukrainian            | koi8u_general_ci    |      1 |
| gb2312   | GB2312 Simplified Chinese   | gb2312_chinese_ci   |      2 |
| greek    | ISO 8859-7 Greek            | greek_general_ci    |      1 |
| cp1250   | Windows Central European    | cp1250_general_ci   |      1 |
| gbk      | GBK Simplified Chinese      | gbk_chinese_ci      |      2 |
| latin5   | ISO 8859-9 Turkish          | latin5_turkish_ci   |      1 |
| armscii8 | ARMSCII-8 Armenian          | armscii8_general_ci |      1 |
| utf8     | UTF-8 Unicode               | utf8_general_ci     |      3 |
| ucs2     | UCS-2 Unicode               | ucs2_general_ci     |      2 |
| cp866    | DOS Russian                 | cp866_general_ci    |      1 |
| keybcs2  | DOS Kamenicky Czech-Slovak  | keybcs2_general_ci  |      1 |
| macce    | Mac Central European        | macce_general_ci    |      1 |
| macroman | Mac West European           | macroman_general_ci |      1 |
| cp852    | DOS Central European        | cp852_general_ci    |      1 |
| latin7   | ISO 8859-13 Baltic          | latin7_general_ci   |      1 |
| utf8mb4  | UTF-8 Unicode               | utf8mb4_general_ci  |      4 |
| cp1251   | Windows Cyrillic            | cp1251_general_ci   |      1 |
| utf16    | UTF-16 Unicode              | utf16_general_ci    |      4 |
| utf16le  | UTF-16LE Unicode            | utf16le_general_ci  |      4 |
| cp1256   | Windows Arabic              | cp1256_general_ci   |      1 |
| cp1257   | Windows Baltic              | cp1257_general_ci   |      1 |
| utf32    | UTF-32 Unicode              | utf32_general_ci    |      4 |
| binary   | Binary pseudo charset       | binary              |      1 |
| geostd8  | GEOSTD8 Georgian            | geostd8_general_ci  |      1 |
| cp932    | SJIS for Windows Japanese   | cp932_japanese_ci   |      2 |
| eucjpms  | UJIS for Windows Japanese   | eucjpms_japanese_ci |      3 |
+----------+-----------------------------+---------------------+--------+
40 rows in set (0.00 sec)

```
其中的Default collation列表示这种字符集中一种默认的比较规则。大家注意返回结果中的最后一列Maxlen，它代表该种字符集表示一个字符最多需要几个字节。

### 比较规则的查看
```
MariaDB [(none)]> show collation like 'utf8_%';
+------------------------------+---------+-----+---------+----------+---------+
| Collation                    | Charset | Id  | Default | Compiled | Sortlen |
+------------------------------+---------+-----+---------+----------+---------+
| utf8_general_ci              | utf8    |  33 | Yes     | Yes      |       1 |
| utf8_bin                     | utf8    |  83 |         | Yes      |       1 |
| utf8_unicode_ci              | utf8    | 192 |         | Yes      |       8 |
| utf8_icelandic_ci            | utf8    | 193 |         | Yes      |       8 |
| utf8_latvian_ci              | utf8    | 194 |         | Yes      |       8 |
| utf8_romanian_ci             | utf8    | 195 |         | Yes      |       8 |
| utf8_slovenian_ci            | utf8    | 196 |         | Yes      |       8 |
| utf8_polish_ci               | utf8    | 197 |         | Yes      |       8 |
| utf8_estonian_ci             | utf8    | 198 |         | Yes      |       8 |
| utf8_spanish_ci              | utf8    | 199 |         | Yes      |       8 |
| utf8_swedish_ci              | utf8    | 200 |         | Yes      |       8 |
| utf8_turkish_ci              | utf8    | 201 |         | Yes      |       8 |
| utf8_czech_ci                | utf8    | 202 |         | Yes      |       8 |
| utf8_danish_ci               | utf8    | 203 |         | Yes      |       8 |
| utf8_lithuanian_ci           | utf8    | 204 |         | Yes      |       8 |
| utf8_slovak_ci               | utf8    | 205 |         | Yes      |       8 |
| utf8_spanish2_ci             | utf8    | 206 |         | Yes      |       8 |
| utf8_roman_ci                | utf8    | 207 |         | Yes      |       8 |
| utf8_persian_ci              | utf8    | 208 |         | Yes      |       8 |
| utf8_esperanto_ci            | utf8    | 209 |         | Yes      |       8 |
| utf8_hungarian_ci            | utf8    | 210 |         | Yes      |       8 |
| utf8_sinhala_ci              | utf8    | 211 |         | Yes      |       8 |
| utf8_german2_ci              | utf8    | 212 |         | Yes      |       8 |
| utf8_croatian_mysql561_ci    | utf8    | 213 |         | Yes      |       8 |
| utf8_unicode_520_ci          | utf8    | 214 |         | Yes      |       8 |
| utf8_vietnamese_ci           | utf8    | 215 |         | Yes      |       8 |
| utf8_general_mysql500_ci     | utf8    | 223 |         | Yes      |       1 |
| utf8_croatian_ci             | utf8    | 576 |         | Yes      |       8 |
| utf8_myanmar_ci              | utf8    | 577 |         | Yes      |       8 |
| utf8_thai_520_w2             | utf8    | 578 |         | Yes      |       4 |
| utf8mb4_general_ci           | utf8mb4 |  45 | Yes     | Yes      |       1 |
| utf8mb4_bin                  | utf8mb4 |  46 |         | Yes      |       1 |
| utf8mb4_unicode_ci           | utf8mb4 | 224 |         | Yes      |       8 |
| utf8mb4_icelandic_ci         | utf8mb4 | 225 |         | Yes      |       8 |
| utf8mb4_latvian_ci           | utf8mb4 | 226 |         | Yes      |       8 |
| utf8mb4_romanian_ci          | utf8mb4 | 227 |         | Yes      |       8 |
| utf8mb4_slovenian_ci         | utf8mb4 | 228 |         | Yes      |       8 |
| utf8mb4_polish_ci            | utf8mb4 | 229 |         | Yes      |       8 |
| utf8mb4_estonian_ci          | utf8mb4 | 230 |         | Yes      |       8 |
| utf8mb4_spanish_ci           | utf8mb4 | 231 |         | Yes      |       8 |
| utf8mb4_swedish_ci           | utf8mb4 | 232 |         | Yes      |       8 |
| utf8mb4_turkish_ci           | utf8mb4 | 233 |         | Yes      |       8 |
| utf8mb4_czech_ci             | utf8mb4 | 234 |         | Yes      |       8 |
| utf8mb4_danish_ci            | utf8mb4 | 235 |         | Yes      |       8 |
| utf8mb4_lithuanian_ci        | utf8mb4 | 236 |         | Yes      |       8 |
| utf8mb4_slovak_ci            | utf8mb4 | 237 |         | Yes      |       8 |
| utf8mb4_spanish2_ci          | utf8mb4 | 238 |         | Yes      |       8 |
| utf8mb4_roman_ci             | utf8mb4 | 239 |         | Yes      |       8 |
| utf8mb4_persian_ci           | utf8mb4 | 240 |         | Yes      |       8 |
| utf8mb4_esperanto_ci         | utf8mb4 | 241 |         | Yes      |       8 |
| utf8mb4_hungarian_ci         | utf8mb4 | 242 |         | Yes      |       8 |
| utf8mb4_sinhala_ci           | utf8mb4 | 243 |         | Yes      |       8 |
| utf8mb4_german2_ci           | utf8mb4 | 244 |         | Yes      |       8 |
| utf8mb4_croatian_mysql561_ci | utf8mb4 | 245 |         | Yes      |       8 |
| utf8mb4_unicode_520_ci       | utf8mb4 | 246 |         | Yes      |       8 |
| utf8mb4_vietnamese_ci        | utf8mb4 | 247 |         | Yes      |       8 |
| utf8mb4_croatian_ci          | utf8mb4 | 608 |         | Yes      |       8 |
| utf8mb4_myanmar_ci           | utf8mb4 | 609 |         | Yes      |       8 |
| utf8mb4_thai_520_w2          | utf8mb4 | 610 |         | Yes      |       4 |
+------------------------------+---------+-----+---------+----------+---------+
59 rows in set (0.00 sec)

```
规律如下
- 比较规则名称以与其关联的字符集的名称开头。如上图的查询结果的比较规则名称都是以utf8开头的。
- 后边紧跟着该比较规则主要作用于哪种语言，比如utf8_polish_ci表示以波兰语的规则比较，utf8_spanish_ci是以西班牙语的规则比较，utf8_general_ci是一种通用的比较规则。
- 名称后缀意味着该比较规则是否区分语言中的重音、大小写啥的，具体可以用的值如下：

|后缀 |	英文释义| 	描述|
|---|---|---|
|_ai |	accent insensitive |	不区分重音|
|_as |	accent sensitive |	区分重音|
|_ci |	case insensitive |	不区分大小写|
|_cs |	case sensitive |	区分大小写|
|_bin| 	binary |	以二进制方式比较|

### 各级别的字符集和比较规则
MySQL有4个级别的字符集和比较规则，分别是：

-    服务器级别
-    数据库级别
-    表级别
-    列级别
#### 服务器级别

|系统变量 |	描述|
|---|---|
|character_set_server |	服务器级别的字符集|
|collation_server |	服务器级别的比较规则|

```
MariaDB [(none)]> show variables like 'character_set%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | latin1                     |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | latin1                     |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)

MariaDB [(none)]> show variables like 'collation%';
+----------------------+-------------------+
| Variable_name        | Value             |
+----------------------+-------------------+
| collation_connection | utf8_general_ci   |
| collation_database   | latin1_swedish_ci |
| collation_server     | latin1_swedish_ci |
+----------------------+-------------------+
3 rows in set (0.00 sec)

```
#### 数据库级别
如果想查看当前数据库使用的字符集和比较规则，可以查看下面两个系统变量的值
（前提是使用USE语句选择当前默认数据库，如果没有默认数据库，则变量与相应的服务器级系统变量具有相同的值）

|系统变量 |	描述|
|---|---|
|character_set_database |	当前数据库的字符集|
|collation_database |	当前数据库的比较规则|

```
MariaDB [(none)]> create database charset_demo_db
    -> character set gb2312
    -> collate gb2312_chinese_ci;
Query OK, 1 row affected (0.02 sec)

MariaDB [(none)]> show variables like 'character_set_database%';
+------------------------+--------+
| Variable_name          | Value  |
+------------------------+--------+
| character_set_database | latin1 |
+------------------------+--------+
1 row in set (0.01 sec)

MariaDB [(none)]> use charset_demo_db
Database changed
MariaDB [charset_demo_db]> show variables like 'character_set_database%';
+------------------------+--------+
| Variable_name          | Value  |
+------------------------+--------+
| character_set_database | gb2312 |
+------------------------+--------+
1 row in set (0.00 sec)

```
可以看到这个charset_demo_db数据库的字符集和比较规则就是我们在创建语句中指定的。
需要注意的一点是： character_set_database 和 collation_database 这两个系统变量是只读的，我们不能通过修改这两个变量的值而改变当前数据库的字符集和比较规则。

数据库的创建语句中也可以不指定字符集和比较规则，比如这样：```CREATE DATABASE 数据库名;```
这样的话将使用服务器级别的字符集和比较规则作为数据库的字符集和比较规则。

#### 表级别
```
MariaDB [charset_demo_db]> create table t( col varchar(10)) character set utf8 collate utf8_general_ci;
Query OK, 0 rows affected (0.05 sec)

MariaDB [charset_demo_db]> show create table t;
+-------+------------------------------------------------------------------------------------------+
| Table | Create Table                                                                             |
+-------+------------------------------------------------------------------------------------------+
| t     | CREATE TABLE `t` (
  `col` varchar(10) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

MariaDB [charset_demo_db]> show create database charset_demo_db;
+-----------------+----------------------------------------------------------------------------+
| Database        | Create Database                                                            |
+-----------------+----------------------------------------------------------------------------+
| charset_demo_db | CREATE DATABASE `charset_demo_db` /*!40100 DEFAULT CHARACTER SET gb2312 */ |
+-----------------+----------------------------------------------------------------------------+
1 row in set (0.00 sec)

```
如果创建和修改表的语句中没有指明字符集和比较规则，将使用该表所在数据库的字符集和比较规则作为该表的字符集和比较规则。

```
MariaDB [charset_demo_db]> create table tt( col varchar(10));
Query OK, 0 rows affected (0.01 sec)

MariaDB [charset_demo_db]> show create table tt;
+-------+---------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                |
+-------+---------------------------------------------------------------------------------------------+
| tt    | CREATE TABLE `tt` (
  `col` varchar(10) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=gb2312 |
+-------+---------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

```

#### 列级别

需要注意的是，对于存储字符串的列，同一个表中的不同的列也可以有不同的字符集和比较规则。我们在创建和修改列定义的时候可以指定该列的字符集和比较规则
```
CREATE TABLE 表名(
    列名 字符串类型 [CHARACTER SET 字符集名称] [COLLATE 比较规则名称],
    其他列...
);
ALTER TABLE 表名 MODIFY 列名 字符串类型 [CHARACTER SET 字符集名称] [COLLATE 比较规则名称];
```

```
MariaDB [charset_demo_db]> alter table t modify col varchar(10) character set gbk collate gbk_chinese_ci;
Query OK, 0 rows affected, 1 warning (0.03 sec)    
Records: 0  Duplicates: 0  Warnings: 1

MariaDB [charset_demo_db]> show create table t;
+-------+------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                               |
+-------+------------------------------------------------------------------------------------------------------------+
| t     | CREATE TABLE `t` (
  `col` varchar(10) CHARACTER SET gbk DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------+------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

```
对于某个列来说，如果在创建和修改的语句中没有指明字符集和比较规则，将使用该列所在表的字符集和比较规则作为该列的字符集和比较规则。
比方说表t的字符集是utf8，比较规则是utf8_general_ci，修改列col的语句是这么写的：
```ALTER TABLE t MODIFY col VARCHAR(10);```
那列col的字符集和编码将使用表t的字符集和比较规则，也就是utf8和utf8_general_ci。

小贴士： 在转换列的字符集时需要注意，如果转换前列中存储的数据不能用转换后的字符集进行表示会发生错误。
比方说原先列使用的字符集是utf8，列中存储了一些汉字，现在把列的字符集转换为ascii的话就会出错，因为ascii字符集并不能表示汉字字符。 

### 仅修改字符集或仅修改比较规则
由于字符集和比较规则是互相有联系的，如果我们只修改了字符集，比较规则也会跟着变化，如果只修改了比较规则，字符集也会跟着变化，具体规则如下：

-    只修改字符集，则比较规则将变为修改后的字符集默认的比较规则。
-    只修改比较规则，则字符集将变为修改后的比较规则对应的字符集。

### 客户端和服务器通信中的字符集
#### 编码和解码使用的字符集不一致的后果
我们知道字符'我'在utf8字符集编码下的字节串长这样：0xE68891，如果一个程序把这个字节串发送到另一个程序里，另一个程序用不同的字符集去解码这个字节串，假设使用的是gbk字符集来解释这串字节，解码过程就是这样的：
-    首先看第一个字节0xE6，它的值大于0x7F（十进制：127），说明是两字节编码，继续读一字节后是0xE688，然后从gbk编码表中查找字节为0xE688对应的字符，发现是字符'鎴'
-    继续读一个字节0x91，它的值也大于0x7F，再往后读一个字节发现木有了，所以这是半个字符。
-    所以0xE68891被gbk字符集解释成一个字符'鎴'和半个字符。

假设用iso-8859-1，也就是latin1字符集去解释这串字节，解码过程如下：
-    先读第一个字节0xE6，它对应的latin1字符为æ。
-    再读第二个字节0x88，它对应的latin1字符为ˆ。
-    再读第二个字节0x91，它对应的latin1字符为‘。
-    所以整串字节0xE68891被latin1字符集解释后的字符串就是'æˆ‘'

可见，如果对于同一个字符串编码和解码使用的字符集不一样，会产生意想不到的结果，作为人类的我们看上去就像是产生了乱码一样。

#### 字符集转换的概念
如果接收0xE68891这个字节串的程序按照utf8字符集进行解码，然后又把它按照gbk字符集进行编码，
最后编码后的字节串就是0xCED2，我们把这个过程称为字符集的转换，也就是字符串'我'从utf8字符集转换为gbk字符集。

#### MySQL中字符集的转换

我们知道从客户端发往服务器的请求本质上就是一个字符串，服务器向客户端返回的结果本质上也是一个字符串，而字符串其实是使用某种字符集编码的二进制数据。
这个字符串可不是使用一种字符集的编码方式一条道走到黑的，从发送请求到返回结果这个过程中伴随着多次字符集的转换，在这个过程中会用到3个系统变量，我们先把它们写出来看一下：

|系统变量 |	描述|
|---|---|
|character_set_client |	服务器解码请求时使用的字符集|
|character_set_connection |	服务器处理请求时会把请求字符串从character_set_client转为character_set_connection|
|character_set_results |	服务器向客户端返回数据时使用的字符集|

一般情况下客户端所使用的字符集和当前操作系统一致，不同操作系统使用的字符集可能不一样，如下：
-    类Unix系统使用的是utf8
-    Windows使用的是gbk

请求从发送到结果返回过程中字符集的变化

-    服务器认为客户端发送过来的请求是用character_set_client编码的。
    假设你的客户端采用的字符集和 character_set_client 不一样的话，这就会出现意想不到的情况。比如我的客户端使用的是utf8字符集，如果把系统变量character_set_client的值设置为ascii的话，服务器可能无法理解我们发送的请求，更别谈处理这个请求了。

-    服务器将把得到的结果集使用character_set_results编码后发送给客户端。
    假设你的客户端采用的字符集和 character_set_results 不一样的话，这就可能会出现客户端无法解码结果集的情况，结果就是在你的屏幕上出现乱码。比如我的客户端使用的是utf8字符集，如果把系统变量character_set_results的值设置为ascii的话，可能会产生乱码。

-    character_set_connection只是服务器在将请求的字节串从character_set_client转换为character_set_connection时使用，它是什么其实没多重要，但是一定要注意，该字符集包含的字符范围一定涵盖请求中的字符，要不然会导致有的字符无法使用character_set_connection代表的字符集进行编码。比如你把character_set_client设置为utf8，把character_set_connection设置成ascii，那么此时你如果从客户端发送一个汉字到服务器，那么服务器无法使用ascii字符集来编码这个汉字，就会向用户发出一个警告。


#### my.conf
```
[server]
skip-networking
default-storage-engine=InnoDB
character-set-server=utf8
collation-server=utf8_general_ci

[client]
default-character-set=utf8

```