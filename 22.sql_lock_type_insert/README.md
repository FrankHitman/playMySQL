# sql lock type insert
![](sql_lock_type_insert.png)

文檔中錯誤處：
    
    CREATE TABLE horse (
        number INT PRIMARY KEY,
        horse_name VARCHAR(100),
        FOREIGN KEY (number) REFERENCES hero(number)
    )Engine=InnoDB CHARSET=utf8;
    
    mysql> begin;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> INSERT INTO horse VALUES(5, '绝影');
    ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`test`.`horse`, CONSTRAINT `horse_ibfk_1` FOREIGN KEY (`number`) REFERENCES `hero` (`number`))
    mysql> INSERT INTO horse VALUES(8, '绝影');
    Query OK, 1 row affected (0.00 sec)
    
    mysql> INSERT INTO horse VALUES(2, '绝影');
    ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`test`.`horse`, CONSTRAINT `horse_ibfk_1` FOREIGN KEY (`number`) REFERENCES `hero` (`number`))
    mysql> 
