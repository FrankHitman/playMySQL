# MySQL數據目錄

## 如何確定mysql中的數據目錄？
    mysql> show variables like '%datadir%';
    +---------------+-----------------+
    | Variable_name | Value           |
    +---------------+-----------------+
    | datadir       | /var/lib/mysql/ |
    +---------------+-----------------+
    1 row in set (0.00 sec)
與docker啟動配置中的數據目錄掛載一致
```shell script
docker run \
 -p 3306:3306 \
 --name mysql5.7 \
 --restart=always \
 -v $PWD/conf:/etc/mysql/conf.d -v $PWD/log:/log -v $PWD/data:/var/lib/mysql -v $PWD/script:/script \
 -e MYSQL_ROOT_PASSWORD=woshidashuaige \
 -d mysql:5.7
```

## 目錄下面有哪些文件？相应的文件的意义是什么？

    $ ll work/siemens/docker/mysql5.7/data/
    total 387152
    drwxr-xr-x@  22 frank  staff   704B Apr  2 15:19 .
    drwxr-xr-x    7 frank  staff   224B Jan 27 11:49 ..
    -rw-r-----@   1 frank  staff    56B Jan 22 12:16 auto.cnf
    -rw-------@   1 frank  staff   1.6K Jan 22 12:16 ca-key.pem
    -rw-r--r--@   1 frank  staff   1.1K Jan 22 12:16 ca.pem
    -rw-r--r--@   1 frank  staff   1.1K Jan 22 12:17 client-cert.pem
    -rw-------@   1 frank  staff   1.6K Jan 22 12:17 client-key.pem
    -rw-r-----@   1 frank  staff   1.3K Jan 22 12:17 ib_buffer_pool
    -rw-r-----@   1 frank  staff    48M Apr  2 15:19 ib_logfile0
    -rw-r-----@   1 frank  staff    48M Jan 22 12:16 ib_logfile1
    -rw-r-----@   1 frank  staff    76M Apr  2 15:19 ibdata1
    -rw-r-----    1 frank  staff    12M Apr  2 15:19 ibtmp1
    drwxr-x---@  77 frank  staff   2.4K Jan 22 12:17 mysql
    drwxr-x---@  90 frank  staff   2.8K Jan 22 12:17 performance_schema
    -rw-------@   1 frank  staff   1.6K Jan 22 12:17 private_key.pem
    -rw-r--r--@   1 frank  staff   452B Jan 22 12:17 public_key.pem
    -rw-r--r--@   1 frank  staff   1.1K Jan 22 12:16 server-cert.pem
    -rw-------@   1 frank  staff   1.6K Jan 22 12:16 server-key.pem
    drwxr-x---@  47 frank  staff   1.5K Jan 22 18:11 siemens_evse_cloud
    drwxr-x---@  25 frank  staff   800B Jan 22 17:09 siemens_evse_job
    drwxr-x---@   9 frank  staff   288B Mar 13 13:53 siemens_time_sheet
    drwxr-x---@ 108 frank  staff   3.4K Jan 22 12:17 sys
    Franks-Mac:~ frank$ 

- 默认/自动生成的SSL和RSA证书和密钥文件。主要是为了客户端和服务器安全通信而创建的一些文件，例如ca-key.pem
- 服务器日志文件。例如ib_logfile0
- 服务器进程文件。
- 系统表空间（system tablespace），默认情况下，InnoDB会在数据目录下创建一个名为ibdata1
```
# 修改系統表空間對應的文件和大小
[server]
innodb_data_file_path=data1:512M;data2:512M:autoextend
```
- 在数据目录下创建一个和数据库名同名的子目录（或者说是文件夹）。
- 在该与数据库名同名的子目录下创建一个名为db.opt的文件，这个文件中包含了该数据库的各种属性，比方说该数据库的字符集和比较规则是个啥。
- 每新建一個表，生成frm文件（表结构的定义）和ibd（表中的数据）文件。可以注意到ibd文件大小都是16kb的整數倍。
- 独立表空间(file-per-table tablespace)，在MySQL5.6.6以及之后的版本中，InnoDB并不会默认的把各个表的数据存储到系统表空间中，
而是为每一个表建立一个独立表空间，例如下面各個ibd文件
```
# 修改配置把數據都存儲到系統表空間，只对新建的表起作用
[server]
innodb_file_per_table=0
```
- 對於視圖（view）只會生成frm文件，例如uv_evse_charger_daily_report.frm


```shell script
Franks-Mac:~ frank$ ll work/siemens/docker/mysql5.7/data/siemens_evse_cloud/
total 33016
drwxr-x---@ 51 frank  staff   1.6K Apr  7 10:43 .
drwxr-xr-x@ 22 frank  staff   704B Apr  3 19:14 ..
-rw-r-----@  1 frank  staff   9.2K Jan 22 17:10 charger.frm
-rw-r-----@  1 frank  staff   144K Jan 22 17:10 charger.ibd
-rw-r-----@  1 frank  staff   8.5K Jan 22 17:10 charger_daily_report.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 charger_daily_report.ibd
-rw-r-----@  1 frank  staff   8.8K Jan 22 17:10 charger_event.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 charger_event.ibd
-rw-r-----@  1 frank  staff   8.4K Jan 22 18:11 charger_event_name.frm
-rw-r-----@  1 frank  staff    96K Jan 22 18:11 charger_event_name.ibd
-rw-r-----@  1 frank  staff   8.7K Jan 22 17:10 charger_log.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 charger_log.ibd
-rw-r-----@  1 frank  staff   8.6K Jan 22 17:10 charger_monthly_report.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 charger_monthly_report.ibd
-rw-r-----@  1 frank  staff   8.6K Jan 22 17:10 charger_status_change_log.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 charger_status_change_log.ibd
-rw-r-----@  1 frank  staff   9.0K Jan 22 17:10 charging_history.frm
-rw-r-----@  1 frank  staff   144K Jan 22 17:10 charging_history.ibd
-rw-r-----@  1 frank  staff   8.7K Jan 22 17:10 charging_schedule.frm
-rw-r-----@  1 frank  staff   128K Jan 22 17:10 charging_schedule.ibd
-rw-r-----@  1 frank  staff   8.7K Jan 22 17:10 charging_station.frm
-rw-r-----@  1 frank  staff    96K Jan 22 17:10 charging_station.ibd
-rw-r-----@  1 frank  staff    61B Jan 22 17:09 db.opt
-rw-r-----@  1 frank  staff   8.8K Jan 22 17:10 electric_vehicle.frm
-rw-r-----@  1 frank  staff   128K Jan 22 17:10 electric_vehicle.ibd
-rw-r-----@  1 frank  staff   8.6K Jan 22 17:10 ev_daily_report.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 ev_daily_report.ibd
-rw-r-----@  1 frank  staff   8.6K Jan 22 17:10 ev_monthly_report.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 ev_monthly_report.ibd
-rw-r-----@  1 frank  staff   8.7K Jan 22 17:10 firmware.frm
-rw-r-----@  1 frank  staff    96K Jan 22 17:10 firmware.ibd
-rw-r-----@  1 frank  staff   8.9K Jan 22 17:10 gateway.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 gateway.ibd
-rw-r-----@  1 frank  staff   8.7K Jan 22 17:10 mobile_user.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 mobile_user.ibd
-rw-r-----@  1 frank  staff   8.5K Jan 22 17:10 network_platform.frm
-rw-r-----@  1 frank  staff    96K Jan 22 17:10 network_platform.ibd
-rw-r-----@  1 frank  staff   8.7K Jan 22 17:10 operator_user.frm
-rw-r-----@  1 frank  staff   112K Jan 22 18:36 operator_user.ibd
-rw-r-----@  1 frank  staff   8.6K Jan 22 18:10 position.frm
-rw-r-----@  1 frank  staff    13M Jan 22 18:11 position.ibd
-rw-r-----@  1 frank  staff   8.7K Jan 22 17:10 private_charger.frm
-rw-r-----@  1 frank  staff   128K Jan 22 17:10 private_charger.ibd
-rw-r-----@  1 frank  staff   8.8K Jan 22 17:10 production_line_charger.frm
-rw-r-----@  1 frank  staff   128K Jan 22 17:10 production_line_charger.ibd
-rw-r-----@  1 frank  staff   8.6K Jan 22 17:10 public_charger.frm
-rw-r-----@  1 frank  staff   112K Jan 22 17:10 public_charger.ibd
-rw-r-----   1 frank  staff   2.5K Apr  7 10:43 uv_evse_charger_daily_report.frm
-rw-r-----   1 frank  staff   2.6K Apr  7 10:43 uv_evse_charger_monthly_report.frm
-rw-r-----   1 frank  staff   2.9K Apr  7 10:43 uv_evse_ev_daily_report.frm
-rw-r-----   1 frank  staff   2.9K Apr  7 10:43 uv_evse_ev_monthly_report.frm
```

## mysql系統數據庫的意義是什麼？
- mysql

这个数据库核心，它存储了MySQL的用户账户和权限信息，一些存储过程、事件的定义信息，一些运行过程中产生的日志信息，一些帮助信息以及时区信息等。
- performance_schema
 
 这个数据库里主要保存MySQL服务器运行过程中的一些状态信息，算是对MySQL服务器的一个性能监控。包括统计最近执行了哪些语句，在执行过程的每个阶段都花费了多长时间，内存的使用情况等等信息。

- information_schema

这个数据库保存着MySQL服务器维护的所有其他数据库的信息，比如有哪些表、哪些视图、哪些触发器、哪些列、哪些索引吧啦吧啦。这些信息并不是真实的用户数据，而是一些描述性信息，有时候也称之为元数据。

- sys

这个数据库主要是通过视图的形式把information_schema和performance_schema结合起来，让程序员可以更方便的了解MySQL服务器的一些性能信息。

但是在mariadb10.4.6版本中沒有見到這個庫。