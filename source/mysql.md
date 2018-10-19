# MySQL57主从复制、备份恢复、故障处理


## 1. MySQL57主从配置


### 1.1. 安装MySQL57
```
yum install mysql57-5.7.17-1.el6.x86_64.rpm
```
### 1.2. 配置MySQL ServerID
```
Master端    修改：
vim /data1/app/services/mysql57/my.cnf   #编辑MySQL配置文件，操作之前先备份一份。
[mysqld]
prot = 3306
server-id = 92    #这里的MySQL主需要修改ServerID就好了，ServerID只是一个标识符。

Slave端修改：
vim /data1/app/services/mysql57/my.cnf  #编辑MySQL配置文件，操作之前先备份一份。
[mysqld]
prot = 3307
server-id = 93    #这里的MySQL也是需要修改ServerID就好了，ServerID只是一个标识符。

启动MySQL：
/etc/init.d/mysql57 start

```
### 1.3. 新增Master授权slave用户
```
mysql -uroot -h127.0.0.1 -P3306
mysql> grant replication slave on *.* to 'shinezone'@'172.16.0.20' identified by 'shinezone2015';
mysql> flush privileges;
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000005 |    31332 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```

### 1.4. Slave端配置MySQL主信息
 ```
 mysql -uroot -hlocalhost -P3307
 mysql> CHANGE MASTER TO MASTER_HOST='172.16.0.18',MASTER_USER='replication',MASTER_PASSWORD='nfH56BmsQ',MASTER_LOG_FILE='mysql-bin.000005',MASTER_LOG_POS=31332,master_port=3306;
 mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.0.18
                  Master_User: shinezone
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 31332
               Relay_Log_File: intranet_ops_yanghongfei03-relay-bin.000005
                Relay_Log_Pos: 650
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 31332
              Relay_Log_Space: 26400
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 91
                  Master_UUID: 800a1d25-b894-11e7-888e-005056b67e19
             Master_Info_File: /data1/app/services/mysql57/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
 ```

## 2. MySQL57模拟1062主键错误处理过程 
### 2.1. Master端新增主键数据表
```
mysql -uroot -hlocalhost -P3307
mysql> create database shinezonetest character set utf8;
mysql> use shinezonetest;
mysql>  create table shinezonetest(id int(2) primary key not null auto_increment,name char(10));
mysql> desc shinezonetest;
+-------+----------+------+-----+---------+----------------+
| Field | Type     | Null | Key | Default | Extra          |
+-------+----------+------+-----+---------+----------------+
| id    | int(2)   | NO   | PRI | NULL    | auto_increment |
| name  | char(10) | YES  |     | NULL    |                |
+-------+----------+------+-----+---------+----------------+
2 rows in set (0.00 sec)
mysql> insert into shinezonetest values('001','yang');  #插入第一条数据test
mysql> select * from shinezonetest ;
+----+------+
| id | name |
+----+------+
|  1 | yang |
+----+------+
1 row in set (0.00 sec)
```
### 2.2. Slave端insert数据模拟故障
```
mysql -uroot -hlocalhost -P3307
mysql> select * from shinezonetest;  #目前同步是一致的。
+----+------+
| id | name |
+----+------+
|  1 | yang |
+----+------+
1 row in set (0.00 sec)
mysql> insert into shinezonetest values('002','hong');  #故意模拟从库插入主键数据
mysql> select * from shinezonetest;
+----+------+
| id | name |
+----+------+
|  1 | yang |
|  2 | hong |
+----+------+
2 rows in set (0.00 sec) 
```
### 2.3. Master同样主库插入冲突的主键数据
```
mysql -uroot -hlocalhost -P3306
mysql> insert into shinezonetest  #这时候已经主键冲突错误了 values('002','hong');
ERROR 1062 (23000): Duplicate entry '2' for key 'PRIMARY'
```
### 2.4. Slave端处理主键冲突错误
```
mysql -uroot -hlocalhost -P3307
mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.0.18
                  Master_User: shinezone
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 31002
               Relay_Log_File: intranet_ops_yanghongfei03-relay-bin.000004
                Relay_Log_Pos: 25345
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1062
                   Last_Error: Error 'Duplicate entry '2' for key 'PRIMARY'' on query. Default database: 'shinezonetest'. Query: 'insert into shinezonetest values('002','hong')'
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 30671
              Relay_Log_Space: 26513
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1062
               Last_SQL_Error: Error 'Duplicate entry '2' for key 'PRIMARY'' on query. Default database: 'shinezonetest'. Query: 'insert into shinezonetest values('002','hong')'
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 91
                  Master_UUID: 800a1d25-b894-11e7-888e-005056b67e19
             Master_Info_File: /data1/app/services/mysql57/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 171026 13:10:02
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)


mysql> stop slave;
mysql> set global sql_slave_skip_counter=1;   ##slave-skip-errors=1062
mysql> start slave;
mysql> show slave status \G;  #这是已经恢复正常
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.0.18
                  Master_User: shinezone
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 31002
               Relay_Log_File: intranet_ops_yanghongfei03-relay-bin.000005
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 31002
              Relay_Log_Space: 26070
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 91
                  Master_UUID: 800a1d25-b894-11e7-888e-005056b67e19
             Master_Info_File: /data1/app/services/mysql57/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

```

## 3. MySQL57备份恢复
MySQL备份方式有两种：  mysqldump默认备份，工具xtartbackup备份（大数据量推荐使用）  
  
  
MySQL恢复方式有三种：  mysqldump默认恢复，工具Xtartbackup恢复，mysqlbinlog数据恢复
### 3.1. mysqldump默认备份恢复

#### 3.1.1. mysqldump备份
```
mysqldump -u 用户名 -p 密码 -h 主机名 -P 端口 数据库名 > 数据库备份名.sql 
mysqldump -uusername -ppassword -Pport -hlocalhost databasename > databasename-`date +%F `.sql
```


#### 3.1.2. mysqldump恢复
```
mysqldump -uusername -ppassword -hlocalhost -Pport databasename < databasename-`date +%F `.sql 
// use databasename;
// source databasename-`date +%F `.sql; 
```
### 3.2. MySQL Xtarbackup工具备份恢复
xtrabackup是Percona公司CTO Vadim参与开发的一款基于InnoDB的在线热备工具，具有开源，免费，支持在线热备，备份恢复速度快，占用磁盘空间小等特点，并且支持不同情况下的多种备份形式。

 xtrabackup的官方下载地址为http://www.percona.com/software/percona-xtrabackup。
xtrabackup包含两个主要的工具，即xtrabackup和innobackupex，二者区别如下：

（1）xtrabackup只能备份innodb和xtradb两种引擎的表，而不能备份myisam引擎的表；

（2）innobackupex是一个封装了xtrabackup的Perl脚本，支持同时备份innodb和myisam，但在对myisam备份时需要加一个全局的读锁。还有就是myisam不支持增量备份。

#### 3.2.1. Innobakupex备份过程
Innobackupex备份过程如下：
![](http://images.cnitblog.com/i/609710/201404/072042369347341.png)
	在图中，备份开始时首先会开启一个后台检测进程，实时检测mysql redo的变化，一旦发现redo中有新的日志写入，立刻将日志记入后台日志文件xtrabackup_log中。之后复制innodb的数据文件和系统表空间文件ibdata1，待复制结束后，执行flush tables with read lock操作，复制.frm，MYI，MYD，等文件（执行flush tableswith read lock的目的是为了防止数据表发生DDL操作，并且在这一时刻获得binlog的位置）最后会发出unlock tables，把表设置为可读可写状态，最终停止xtrabackup_log
	

**主要参数介绍：**

--slave-info：
     它会记录master服务器的binary log的pos和name。会把记录的信息记录在xtrabackup_slave_info

--safe-salve-backup:
         它会停止slave SQL 进程，等备份完后，重新打开slave的SQL进程


--safe-slave-backup-timeout=XXX
备份开始时每隔3s会通过SHOW STATUS LIKE "slave_open_temp_tables"\G来检查当前实例的临时表数量，数量为0则开始备份，否则在尝试100次（默认300s）超时后，stop slave sql_thread开始备份

--force-tar --stream=tar /tmp 
         这些命令是用来压缩备份为tar文件。具体看官方文档

--no-timestamp
使用了这个参数后，将不会建立带时间戳的文件夹，只会直接放在指定的目录中

--socket=socket
指定使用的socket路径
#### 3.2.2. 安装xtarbackup
```
[root@intranet_ops_yanghongfei03 ~]#yum install https://wx.shinezone.com/rpmsoft/shinezone_yum/centos6/x86_64/szpercona-xtrabackup-2.4.8-1.el6.x86_64.rpm
[root@intranet_ops_yanghongfei03 ~]#rpm -ql szpercona-xtrabackup
/data1/software/xtrabackup/bin/innobackupex
/data1/software/xtrabackup/bin/xbcloud
/data1/software/xtrabackup/bin/xbcloud_osenv
/data1/software/xtrabackup/bin/xbcrypt
/data1/software/xtrabackup/bin/xbstream
/data1/software/xtrabackup/bin/xtrabackup
```

#### 3.2.3. 创建备份用户
```
mysql> create user 'backup'@'%' identified by 'yanghongfei';
Query OK, 0 rows affected (0.00 sec)

mysql> grant reload,lock tables, replication client,process,super on *.* to 'backup'@'%' identified by 'yanghongfei';
Query OK, 0 rows affected, 1 warning (0.04 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
```
#### 3.2.4. 创建全备份
备份数据存放在/data1/backup/下面，innobackupex会自动创建一个文件夹，是当前系统的时间戳
  
```    
innobackupex --user=backup --password=yanghongfei --socket=/var/lib/mysql/szmysqld.sock /data1/backup/  #默认全备份带时间戳的

innobackupex --user=backup --password=yanghongfei --socket=/var/lib/mysql/szmysqld.sock --no-timestamp /data1/backup/databasename  #--no-timestamp参数不带时间戳

```
#### 3.2.5. Slave备份操作
在slave备份，并在备份时停止主从复制进程，完成后再开启

```
innobackupex --user=backup --password=yanghongfei --rsync --parallel=2 --lock-wait-query-type=all --lock-wait-timeout=300 --lock-wait-threshold=30 --kill-long-query-type=all --kill-long-queries-timeout=30 --socket=/var/lib/mysql/szmysqld.sock /data1/backup
```

#### 3.2.6. 备份单个数据库
```
innobackupex --user=backup --password=yanghongfei --databases='yanghongfei' --socket=/var/lib/mysql/szmysqld.sock --no-timestamp /data1/backup/yanghongfei
```

#### 3.2.7. xtarbackup恢复
```
mysql -uroot 
mysql> DROP DATABASE yanghongfei;
mysql> quit

/etc/init.d/mysql57 stop
cd /data1/app/services/mysql57
mv data data_bak  #线上不要随意删除数据库 #Xtarbackup恢复， 保证data目录是空的
mkdir data
chown -R mysql.mysql data
chmod -R 755 data
innobackupex --defaults-file=/data1/app/services/mysql57/my.cnf --apply-log --redo-only --use-memory=1G 2017-10-27_17-19-24/
innobackupex --defaults-file=/data1/app/services/mysql57/my.cnf  --socket=/var/lib/mysql/szmysql.sock --copy-back /data1/backup/2017-10-27_17-19-24/
```

### 3.3.  MySQL binlog日志恢复

 binlog 基本认识
    MySQL的二进制日志可以说是MySQL最重要的日志了，它记录了所有的DDL和DML(除了数据查询语句)语句，以事件形式记录，还包含语句所执行的消耗的时间，MySQL的二进制日志是事务安全型的。

    一般来说开启二进制日志大概会有1%的性能损耗(参见MySQL官方中文手册 5.1.24版)。二进制有两个最重要的使用场景: 
    其一：MySQL Replication在Master端开启binlog，Mster把它的二进制日志传递给slaves来达到master-slave数据一致的目的。 
    其二：自然就是数据恢复了，通过使用mysqlbinlog工具来使恢复数据。
    
    二进制日志包括两类文件：二进制日志索引文件（文件名后缀为.index）用于记录所有的二进制文件，二进制日志文件（文件名后缀为.00000*）记录数据库所有的DDL和DML(除了数据查询语句)语句事件。 
    
#### 3.3.1. 开启mysql bin-log
```
more /data1/app/services/mysql57/my.cnf

log-bin  = mysql-bin
binlog_format = MIXED   #binlog日志格式，mysql默认采用statement，建议使用mixed
binlog_cache_size = 2M    #binlog使用缓存内存的大小
sync_binlog = 0   
  
  # 当事务提交后，Mysql仅仅是将binlog_cache中的数据写入Binlog文件，但不执行fsync之类的磁盘       同步指令通知文件系统将缓存刷新到磁盘，而让Filesystem自行决定什么时候来做同步，这个是性能最好的。
expire_logs_days = 7   #二进制日志自动删除/过期的天数。默认值为0,表示“没有自动删除”
lower_case_table_names=1   默认为0，大小写敏感。
设置1，大小写不敏感。创建的表，数据库都是以小写形式存放在磁盘上，对于sql语句都是转换为小写对表和DB进行查找。
设置2，创建的表和DB依据语句上格式存放，凡是查找都是转换为小写进行。
```

#### 3.3.2. 查看binlog信息
```
mysql> show variables like 'log_%';
+----------------------------------------+--------------------------------------------------+
| Variable_name                          | Value                                            |
+----------------------------------------+--------------------------------------------------+
| log_bin                                | ON      ##ON表示已开启                                            |
| log_bin_basename                       | /data1/app/services/mysql57/data/mysql-bin       |
| log_bin_index                          | /data1/app/services/mysql57/data/mysql-bin.index |
| log_bin_trust_function_creators        | ON                                               |
| log_bin_use_v1_row_events              | OFF                                              |
| log_builtin_as_identified_by_password  | OFF                                              |
| log_error                              | /data1/app/services/mysql57/log/mysqld.log       |
| log_error_verbosity                    | 3                                                |
| log_output                             | FILE                                             |
| log_queries_not_using_indexes          | ON                                               |
| log_slave_updates                      | OFF                                              |
| log_slow_admin_statements              | OFF                                              |
| log_slow_slave_statements              | OFF                                              |
| log_statements_unsafe_for_binlog       | ON                                               |
| log_syslog                             | OFF                                              |
| log_syslog_facility                    | daemon                                           |
| log_syslog_include_pid                 | ON                                               |
| log_syslog_tag                         |                                                  |
| log_throttle_queries_not_using_indexes | 0                                                |
| log_timestamps                         | UTC                                              |
| log_warnings                           | 2                                                |
+----------------------------------------+--------------------------------------------------+
21 rows in set (0.11 sec)

```
#### 3.3.3. 常用binlog日志操作命令
```
1.查看所有binlog日志列表
mysql> show master logs;

2.查看master状态，即最后(最新)一个binlog日志的编号名称，及其最后一个操作事件pos结束点(Position)值
mysql> show master status;

3.刷新log日志，自此刻开始产生一个新编号的binlog日志文件
mysql> flush logs;
注：每当mysqld服务重启时，会自动执行此命令，刷新binlog日志；在mysqldump备份数据时加 -F 选项也会刷新binlog日志；

4.重置(清空)所有binlog日志
mysql> reset master;
```

#### 3.3.4. binlog日志打开方式
```
1.使用mysqlbinlog自带查看命令法：
注: binlog是二进制文件，普通文件查看器cat more vi等都无法打开，必须使用自带的 mysqlbinlog 命令查看

/data1/app/services/mysql57/bin/mysqlbinlog /data1/app/services/mysql57/data/mysql-bin.000001

2.使用MySQL查询命令进行查询
mysql>  show binlog events in 'mysql-bin.000001';
```

#### 3.3.5. mysqlbinlog恢复
```
1. 机器开启binlog，创建数据库
mysql> create database shinezonetest;
mysql> use shinezonetest;
2. 新建一张表
mysql>         CREATE TABLE IF NOT EXISTS `tt` (
    ->           `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
    ->           `name` varchar(16) NOT NULL,
    ->           `sex` enum('m','w') NOT NULL DEFAULT 'm',
    ->           `age` tinyint(3) unsigned NOT NULL,
    ->           `classid` char(6) DEFAULT NULL,
    ->           PRIMARY KEY (`id`)
    ->          ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.14 sec)

3. 插入数据
mysql> insert into tt(`name`,`sex`,`age`,`classid`) values('yiyi','w',20,'cls1'),('xiaoer','m',22,'cls3'),('zhangsan','w',21,'cls5'),('lisi','m',20,'cls4'),('wangwu','w',26,'cls6');
Query OK, 5 rows affected (0.04 sec)
Records: 5  Duplicates: 0  Warnings: 0

4. 查看目前表数据
mysql> select * from tt;
+----+----------+-----+-----+---------+
| id | name     | sex | age | classid |
+----+----------+-----+-----+---------+
|  1 | yiyi     | w   |  20 | cls1    |
|  2 | xiaoer   | m   |  22 | cls3    |
|  3 | zhangsan | w   |  21 | cls5    |
|  4 | lisi     | m   |  20 | cls4    |
|  5 | wangwu   | w   |  26 | cls6    |
+----+----------+-----+-----+---------+
5 rows in set (0.00 sec)

5. 更新2条语句
mysql> update shinezonetest.tt set name ='李四' where id = '4';
Query OK, 1 row affected (0.03 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> update shinezonetest.tt set name ='小二' where id = '2';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

6. 再次查看表数据
mysql> select * from tt;
+----+----------+-----+-----+---------+
| id | name     | sex | age | classid |
+----+----------+-----+-----+---------+
|  1 | yiyi     | w   |  20 | cls1    |
|  2 | 小二     | m   |  22 | cls3    |
|  3 | zhangsan | w   |  21 | cls5    |
|  4 | 李四     | m   |  20 | cls4    |
|  5 | wangwu   | w   |  26 | cls6    |
+----+----------+-----+-----+---------+
5 rows in set (0.00 sec)

7. 以上由于开启了binlog 所有操作都被记录下来了，接下来模拟故意删除表
mysql> DROP TABLE tt;
Query OK, 0 rows affected (0.09 sec)
mysql> show tables;   #此时表已经被删除了
+-------------------------+
| Tables_in_shinezonetest |
+-------------------------+
| shinezonetest           |
+-------------------------+
1 row in set (0.00 sec)

8. 表已经被误删除了，此时执行一次刷新日志索引操作，重新开始新的binlog日志记录文件，理论说 mysql-bin.000023 这个文件不会再有后续写入了(便于我们分析原因及查找pos点)，以后所有数据库操作都会写入到下一个日志文件；
mysql> flush logs;
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |      2074 |
| mysql-bin.000002 |       154 |
+------------------+-----------+
2 rows in set (0.00 sec)

mysql> show master status;   #可以看出目前binlog已经开始写入到第二个文件中
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

9. Copy binlog日志用于分析问题
cp /data1/app/services/mysql57/data/mysql-bin.000001 ~/

10. mysqlbinlog导出sql文件，删除DROP语句进行souce恢复
mysqlbinlog -vv mysql-bin.000001 > shinezonetest.sql

11. 登录服务器，查看binlog日志详情
mysql> show binlog events in 'mysql-bin.000001';
+------------------+------+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Log_name         | Pos  | Event_type     | Server_id | End_log_pos | Info                                                                                                                                                                                                                                                                                                                                                                          |
+------------------+------+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| mysql-bin.000001 |    4 | Format_desc    |        92 |         123 | Server ver: 5.7.17-log, Binlog ver: 4                                                                                                                                                                                                                                                                                                                                         |
| mysql-bin.000001 |  123 | Previous_gtids |        92 |         154 |                                                                                                                                                                                                                                                                                                                                                                               |
| mysql-bin.000001 |  154 | Anonymous_Gtid |        92 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                                                                                                                                                          |
| mysql-bin.000001 |  219 | Query          |        92 |         655 | use `shinezonetest`; CREATE TABLE IF NOT EXISTS `tt` (
          `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
          `name` varchar(16) NOT NULL,
          `sex` enum('m','w') NOT NULL DEFAULT 'm',
          `age` tinyint(3) unsigned NOT NULL,
          `classid` char(6) DEFAULT NULL,
          PRIMARY KEY (`id`)
         ) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
| mysql-bin.000001 |  655 | Anonymous_Gtid |        92 |         720 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                                                                                                                                                          |
| mysql-bin.000001 |  720 | Query          |        92 |         817 | BEGIN                                                                                                                                                                                                                                                                                                                                                                         |
| mysql-bin.000001 |  817 | Intvar         |        92 |         849 | INSERT_ID=1                                                                                                                                                                                                                                                                                                                                                                   |
| mysql-bin.000001 |  849 | Query          |        92 |        1114 | use `shinezonetest`; insert into tt(`name`,`sex`,`age`,`classid`) values('yiyi','w',20,'cls1'),('xiaoer','m',22,'cls3'),('zhangsan','w',21,'cls5'),('lisi','m',20,'cls4'),('wangwu','w',26,'cls6')                                                                                                                                                                            |
| mysql-bin.000001 | 1114 | Xid            |        92 |        1145 | COMMIT /* xid=36 */                                                                                                                                                                                                                                                                                                                                                           |
| mysql-bin.000001 | 1145 | Anonymous_Gtid |        92 |        1210 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                                                                                                                                                          |
| mysql-bin.000001 | 1210 | Query          |        92 |        1307 | BEGIN                                                                                                                                                                                                                                                                                                                                                                         |
| mysql-bin.000001 | 1307 | Query          |        92 |        1456 | use `shinezonetest`; update shinezonetest.tt set name ='李四' where id = '4'                                                                                                                                                                                                                                                                                                  |
| mysql-bin.000001 | 1456 | Xid            |        92 |        1487 | COMMIT /* xid=40 */                                                                                                                                                                                                                                                                                                                                                           |
| mysql-bin.000001 | 1487 | Anonymous_Gtid |        92 |        1552 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                                                                                                                                                          |
| mysql-bin.000001 | 1552 | Query          |        92 |        1649 | BEGIN                                                                                                                                                                                                                                                                                                                                                                         |
| mysql-bin.000001 | 1649 | Query          |        92 |        1798 | use `shinezonetest`; update shinezonetest.tt set name ='小二' where id = '2'                                                                                                                                                                                                                                                                                                  |
| mysql-bin.000001 | 1798 | Xid            |        92 |        1829 | COMMIT /* xid=41 */                                                                                                                                                                                                                                                                                                                                                           |
| mysql-bin.000001 | 1829 | Anonymous_Gtid |        92 |        1894 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                                                                                                                                                          |
| mysql-bin.000001 | 1894 | Query          |        92 |        2027 | use `shinezonetest`; DROP TABLE `tt` /* generated by server */                                                                                                                                                                                                                                                                                                                |
| mysql-bin.000001 | 2027 | Rotate         |        92 |        2074 | mysql-bin.000002;pos=4                                                                                                                                                                                                                                                                                                                                                        |
+------------------+------+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
20 rows in set (0.00 sec)

mysql>

 通过分析查看造成DROP TABLE 的语句在 1894 - 2027之间。
 
12. 通过Pos点进行部分恢复(恢复DROP语句之前所有数据)
/data1/app/services/mysql57/bin/mysqlbinlog --stop-position=1894 --database=shinezonetest  /data1/app/services/mysql57/data/mysql-bin.000001  | mysql -uroot -P3307 -v shinezonetest
mysql> select * from tt;  #可以看到此数据表已经被恢复，且数据在
+----+----------+-----+-----+---------+
| id | name     | sex | age | classid |
+----+----------+-----+-----+---------+
|  1 | yiyi     | w   |  20 | cls1    |
|  2 | 小二     | m   |  22 | cls3    |
|  3 | zhangsan | w   |  21 | cls5    |
|  4 | 李四     | m   |  20 | cls4    |
|  5 | wangwu   | w   |  26 | cls6    |
+----+----------+-----+-----+---------+
5 rows in set (0.00 sec)

13. mysqlbinlog根据pos点按照事物区间单独恢复(恢复到Update语句之前)
/data1/app/services/mysql57/bin/mysqlbinlog  --start-position=219 --stop-position=1307 --database=shinezonetest  /data1/app/services/mysql57/data/mysql-bin.000001  | mysql -uroot -P3307 -v shinezonetest
mysql> select * from tt;  #可以看出来以下查询信息为Update数据
+----+----------+-----+-----+---------+
| id | name     | sex | age | classid |
+----+----------+-----+-----+---------+
|  1 | yiyi     | w   |  20 | cls1    |
|  2 | xiaoer   | m   |  22 | cls3    |
|  3 | zhangsan | w   |  21 | cls5    |
|  4 | lisi     | m   |  20 | cls4    |
|  5 | wangwu   | w   |  26 | cls6    |
+----+----------+-----+-----+---------+
5 rows in set (0.00 sec)

14. 根据时间点进行恢复（DROP语句之前的数据）
# --stop-datetime="2017-10-30 10:23:13"  #恢复此时间点之前的数据

/data1/app/services/mysql57/bin/mysqlbinlog  --stop-datetime="2017-10-30 10:23:13"  --database=shinezonetest  /data1/app/services/mysql57/data/mysql-bin.000001  | mysql -uroot -P3307 -v shinezonetest


15. 根据时间点进行恢复（Update之前的数据）
#--start-datetime="2017-10-30 10:13:49"
#--stop-datetime="2017-10-30 10:21:32"

/data1/app/services/mysql57/bin/mysqlbinlog --start-datetime="2017-10-30 10:13:49"  --stop-datetime="2017-10-30 10:21:32" --database=shinezonetest  /data1/app/services/mysql57/data/mysql-bin.000001  | mysql -uroot -P3307 -v shinezonetest
mysql> select * from tt;  #可以看出来以下查询信息为Update数据
+----+----------+-----+-----+---------+
| id | name     | sex | age | classid |
+----+----------+-----+-----+---------+
|  1 | yiyi     | w   |  20 | cls1    |
|  2 | xiaoer   | m   |  22 | cls3    |
|  3 | zhangsan | w   |  21 | cls5    |
|  4 | lisi     | m   |  20 | cls4    |
|  5 | wangwu   | w   |  26 | cls6    |
+----+----------+-----+-----+---------+
5 rows in set (0.00 sec)

 总结：恢复，就是让mysql将保存在binlog日志中指定段落区间的sql语句逐个重新执行一次。
```








