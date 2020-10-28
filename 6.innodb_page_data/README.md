# innodb page data

## File Header

名称	|占用空间大小|	描述
----|----|----
FIL_PAGE_SPACE_OR_CHKSUM	|4字节|	页的校验和（checksum值）
FIL_PAGE_OFFSET	|4字节|	页号
FIL_PAGE_PREV	|4字节|	上一个页的页号
FIL_PAGE_NEXT	|4字节|	下一个页的页号
FIL_PAGE_LSN	|8字节|	页面被最后修改时对应的日志序列位置（英文名是：Log Sequence Number）
FIL_PAGE_TYPE	|2字节|	该页的类型
FIL_PAGE_FILE_FLUSH_LSN	|8字节|	仅在系统表空间的一个页中定义，代表文件至少被刷新到了对应的LSN值
FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID	|4字节|	页属于哪个表空间

- FIL_PAGE_OFFSET，表空间中的每一个页都对应着一个页号，也就是FIL_PAGE_OFFSET，这个页号由4个字节组成，也就是32个比特位，
所以一个表空间最多可以拥有2³²个页，如果按照页的默认大小16KB来算，一个表空间最多支持64TB的数据。
表空间的第一个页的页号为0，之后的页号分别是1，2，3...依此类推

## 獨立表空間結構
### 區（extent）
对于16KB的页来说，连续的64个页就是一个区，占用1M空間。每256个区被划分成一组。

- FREE：空閒的區
- FREE_FRAG：有剩餘空間的碎片區
- FULL_FRAG：沒有剩餘空間的碎片區
- FSEG：附屬於某個段的區

处于FREE、FREE_FRAG以及FULL_FRAG这三种状态的区都是独立的，算是直属于表空间；而处于FSEG状态的区是附属于某个段的。
### 段（segment）
叶子节点有自己独有的区，非叶子节点也有自己独有的区。存放叶子节点的区的集合就算是一个段（segment），存放非叶子节点的区的集合也算是一个段。
也就是说一个索引会生成2个段，一个叶子节点段，一个非叶子节点段。

- 葉子節點段
- 非葉子節點段
- 回滾段
### 碎片區（fragment）
为了考虑以完整的区为单位分配给某个段对于数据量较小的表太浪费存储空间的这种情况，设计InnoDB的大叔们提出了一个碎片（fragment）区的概念，
也就是在一个碎片区中，并不是所有的页都是为了存储同一个段的数据而存在的，而是碎片区中的页可以用于不同的目的，比如有些页用于段A，有些页用于段B，有些页甚至哪个段都不属于。
碎片区直属于表空间，并不属于任何一个段。

为某个段分配存储空间的策略是这样的：

- 在刚开始向表中插入数据的时候，段是从某个碎片区以单个页面为单位来分配存储空间的。
- 当某个段已经占用了32个碎片区页面之后，就会以完整的区为单位来分配存储空间。

所以现在段不能仅定义为是某些区的集合，更精确的应该是某些零散的页面以及一些完整的区的集合。


## 數據字典
InnoDB存储引擎特意定义了一些列的内部系统表（internal system table）来记录这些这些元数据，這些系統表稱為數據字典

表名|	描述
----|----
SYS_TABLES	|整个InnoDB存储引擎中所有的表的信息
SYS_COLUMNS	|整个InnoDB存储引擎中所有的列的信息
SYS_INDEXES	|整个InnoDB存储引擎中所有的索引的信息
SYS_FIELDS	|整个InnoDB存储引擎中所有的索引对应的列的信息
SYS_FOREIGN	|整个InnoDB存储引擎中所有的外键的信息
SYS_FOREIGN_COLS	|整个InnoDB存储引擎中所有的外键对应列的信息
SYS_TABLESPACES	|整个InnoDB存储引擎中所有的表空间信息
SYS_DATAFILES	|整个InnoDB存储引擎中所有的表空间对应文件系统的文件路径信息
SYS_VIRTUAL	|整个InnoDB存储引擎中所有的虚拟生成列的信息

這些表在information_schema庫中

    | INNODB_SYS_INDEXES                    |
    | INNODB_SYS_TABLES                     |
    | INNODB_SYS_FIELDS                     |
    | INNODB_SYS_COLUMNS                    |
    | INNODB_SYS_FOREIGN                    |
    | INNODB_SYS_TABLESTATS                 |

在information_schema数据库中的这些以INNODB_SYS开头的表并不是真正的内部系统表（内部系统表就是我们上边唠叨的以SYS开头的那些表），
而是在存储引擎启动时读取这些以SYS开头的系统表，然后填充到这些以INNODB_SYS开头的表中。
以INNODB_SYS开头的表和以SYS开头的表中的字段并不完全一样，但供大家参考已经足矣。

## 總結
![](summary.jpg)