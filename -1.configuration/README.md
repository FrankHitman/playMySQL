### the order to load config file
在类UNIX操作系统中，MySQL会按照下列路径来寻找配置文件：

|路径名 |	备注|
|---|---|
|/etc/my.cnf| 	|
|/etc/mysql/my.cnf| 	|
|SYSCONFDIR/my.cnf 	||
|$MYSQL_HOME/my.cnf |	特定于服务器的选项（仅限服务器）|
|defaults-extra-file |	命令行指定的额外配置文件路径|
|~/.my.cnf |	用户特定选项|
|~/.mylogin.cnf |	用户特定的登录路径选项（仅限客户端）|

The later file will cover the earlier file's config.

In config file the later config group will cover the earlier config group.

The config after init command will cover the config in the file.

```mysqld --defaults-file=/tmp/myconfig.txt``` this config will replace the config file above.

### init command and their config group
配置文件中不同的选项组是给不同的启动命令使用的，如果选项组名称与程序名称相同，则组中的选项将专门应用于该程序。
例如， [mysqld]和[mysql]组分别应用于mysqld服务器程序和mysql客户端程序。不过有两个选项组比较特别：

-    [server]组下边的启动选项将作用于所有的服务器程序。
-    [client]组下边的启动选项将作用于所有的客户端程序。

|启动命令| 	类别| 	能读取的组|
|---|---|---|
|mysqld 	|启动服务器 	|[mysqld]、[server]|
|mysqld_safe| 	启动服务器| 	[mysqld]、[server]、[mysqld_safe]|
|mysql.server| 	启动服务器| 	[mysqld]、[server]、[mysql.server]|
|mysql| 	启动客户端| 	[mysql]、[client]|
|mysqladmin| 	启动客户端| 	[mysqladmin]、[client]|
|mysqldump| 	启动客户端 |	[mysqldump]、[client]|

### config after init command 
```mysqld -P3307``` == ```mysqld -P 3307```

|长形式 |	短形式 |	含义|
|---|---|---|
|--host| 	-h |	主机名|
|--user |	-u |	用户名|
|--password |	-p |	密码|
|--port |	-P |	端口|
|--version |	-V |	版本信息|

### 系统变量的作用范围的概念，具体来说作用范围分为这两种：

-    GLOBAL：全局变量，影响服务器的整体操作。
-    SESSION：会话变量，影响某个客户端连接的操作。（注：SESSION有个别名叫LOCAL）

还是以default_storage_engine举例，在服务器启动时会初始化一个名为default_storage_engine，作用范围为GLOBAL的系统变量。
之后每当有一个客户端连接到该服务器时，服务器都会单独为该客户端分配一个名为default_storage_engine，作用范围为SESSION的系统变量，
该作用范围为SESSION的系统变量值按照当前作用范围为GLOBAL的同名系统变量值进行初始化。

```
语句一：SET GLOBAL default_storage_engine = MyISAM;
语句二：SET @@GLOBAL.default_storage_engine = MyISAM;

如果只想对本客户端生效，也可以选择下边三条语句中的任意一条来进行设置：
语句一：SET SESSION default_storage_engine = MyISAM;
语句二：SET @@SESSION.default_storage_engine = MyISAM;
语句三：SET default_storage_engine = MyISAM;
```
查看不同作用范围的系统变量
```
SHOW [GLOBAL|SESSION] VARIABLES [LIKE 匹配的模式];
show variables like 'default_storage_engine';  # default show is session
```
如果某个客户端改变了某个系统变量在`GLOBAL`作用范围的值，并不会影响该系统变量在当前已经连接的客户端作用范围为`SESSION`的值，
只会影响后续连入的客户端在作用范围为`SESSION`的值。 


### 状态变量
为了让我们更好的了解服务器程序的运行情况，MySQL服务器程序中维护了好多关于程序运行状态的变量，它们被称为状态变量。
比方说Threads_connected表示当前有多少客户端与服务器建立了连接，Handler_update表示已经更新了多少行记录吧啦吧啦
```
MariaDB [(none)]> show status like 'thread%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Threadpool_idle_threads | 0     |
| Threadpool_threads      | 0     |
| Threads_cached          | 0     |
| Threads_connected       | 1     |
| Threads_created         | 1     |
| Threads_running         | 1     |
+-------------------------+-------+

```

