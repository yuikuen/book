# MySQL Master-Slave

> MySQL 主从复制的方式有多种，本文主要介绍基于日志 binlog 的主从复制方式

## 概要说明

1. **MySQL 主从复制（也称 A/B 复制） 的原理**

- Master 将数据改变记录到二进制日志 `binary log` 中(配置文件 `log-bin` 指定的文件，叫做二进制事件 `binary log events`)
- Slave 通过 I/O 线程读取 Master 中的 `binary log events` 并写入到它的中继日志 `relay log`
- Slave 重做中继日志中的事件，把中继日志中的事件信息逐条在本地执行，完成数据在本地的存储，从而实现数据重放

2. **主从配置注意事项**

- 主从服务器操作系统版本和位数一致；
- Master 和 Slave 数据库的版本要一致；
- Master 和 Slave 数据库中的数据要一致；
- Master 开启二进制日志，Master 和 Slave 的 Server_id 在局域网内必须唯一；

3. **主从配置操作步骤**

- Master 配置
  - 安装数据库
  - 修改数据库配置文件，指明 Server_id，开启二进制日志 `log-bin`
  - 启动数据库，查看当前是哪个日志，`Position` 号是多少
  - 登录数据库，授权数据复制用户(IP 地址为从机 IP 地址，如双向主从，需授权本机 IP 地址，此时自己的 IP 地址就是从 IP 地址)
  - 备份数据库(记得加锁和解锁)
  - 传送备份数据到 Slave 上；
  - 启动数据库
- Slave 配置
  - 安装数据库
  - 修改数据库配置文件，指明 Server_id(如双向主从，也需开启二进制日志 `log-bin`)
  - 启动数据库，还原备份；
  - 查看当前是哪个日志，`Positiion` 号是多少(单向主从此步不需要，双向主从需要)
  - 指定 Master 地址、用户、密码等信息
  - 开户同步，查看状态

## 实验操作

- Master IP：188.188.4.210
- Slave    IP：188.188.4.211
- MySQL 版本：MySQL 5.7.37

### Master 配置

1）编辑主节点数据库配置文件

**日志格式**指二进制日志的三种格式

- binlog_format=statement
- binlog_format=rox
- binlog_format=mixed

```bash
$ vim /usr/local/mysql/my.cnf          # 一般默认为/etc/my.cnf，具体以实际环境为准
[mysqld]
lower_case_table_names=1               # 表名不区分大小写
binlog_format=MIXED                    # 复制模式，日志格式

server-id=1                            # [必须]服务器唯一ID,默认是1，多主从时注意不可重复
log-bin=mysql-bin                      # 开启二进制日志，可指定存储位置，默认mysql目录
log-bin-index=mysql-bin.index          # 打开二进制日志文件索引

//以下非必须项，仅供参考
sync_binlog=1                          # 控制数据库binlog刷到磁盘上，0不控制，性能最好，1每次事务提交都会刷到日志文件中，性能最差但最安全
expire_logs_days=7                     # 保留天数，以防日志占满磁盘，默认为0，表示不自动删除

max_binlog_size=100m                   # binlog每个日志文件大小
binlog_cache_size=4m                   # binlog缓存大小
max_binlog_cache_size=512m             # 最大binlog缓存大小

binlog-do-db=repldb                    # 需同步的二进制数据库名，可配置多个
binlog-ignore-db=mysql                 # 不同步的数据库，可配置多个
binlog-ignore-db=information_schema    
binlog-ignore-db=performation_schema
binlog-ignore-db=sys

auto-increment-offset=1                # 表中自增字段每次的偏移量
auto-increment-increment=1             # 表中自增字段每次的自增量
slave-skip-errors=all                  # 跳过从库错误
```

**注意：**每次事务提交，MySQL 都会把 binlog 刷下去，是最安全的，但性能损耗最大。这样配置的话，在数据库所在的主机操作系统损坏或突然掉电的情况下，系统才有可能丢失 1 个事务的数据。binlog 是顺序 IO，虽然 sync_binlog=1，多个事务同时提交，但同样很大影响MySQL 的 IO 性能，按需设置。

2）重启加载配置，查看是否正常生成 `log-bin` 文件

```bash
$ systemctl restart mysql && systemctl status mysql

# 重启后，可在data目录下看到，配置文件中定义的log-bin=mysql-bin开头的文件
$ ll ./data/
total 123100
-rw-r----- 1 mysql mysql       56 Apr 14 15:33 auto.cnf
-rw------- 1 mysql mysql     1680 Apr 14 15:33 ca-key.pem
-rw-r--r-- 1 mysql mysql     1112 Apr 14 15:33 ca.pem
-rw-r--r-- 1 mysql mysql     1112 Apr 14 15:33 client-cert.pem
-rw------- 1 mysql mysql     1680 Apr 14 15:33 client-key.pem
-rw-r----- 1 mysql mysql      289 Apr 15 17:03 ib_buffer_pool
-rw-r----- 1 mysql mysql 12582912 Apr 15 17:03 ibdata1
-rw-r----- 1 mysql mysql 50331648 Apr 15 17:03 ib_logfile0
-rw-r----- 1 mysql mysql 50331648 Apr 14 15:33 ib_logfile1
-rw-r----- 1 mysql mysql 12582912 Apr 15 17:03 ibtmp1
drwxr-x--- 2 mysql mysql     4096 Apr 14 15:33 mysql
-rw-r----- 1 mysql mysql      177 Apr 15 16:59 mysql-bin.000001
-rw-r----- 1 mysql mysql      177 Apr 15 17:01 mysql-bin.000002
-rw-r----- 1 mysql mysql      177 Apr 15 17:03 mysql-bin.000003
-rw-r----- 1 mysql mysql      154 Apr 15 17:03 mysql-bin.000004
-rw-r----- 1 mysql mysql       76 Apr 15 17:03 mysql-bin.index
```

3）查看日志信息

```sql
$ mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.37-log Source distribution

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
# 查看二进制日志是否开启
mysql> show global variables like '%log%';
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220415170520671.png)

```sql
# 查看主节点二进制日志列表
mysql> SHOW MASTER LOGS;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       177 |
| mysql-bin.000003 |       177 |
| mysql-bin.000004 |       154 |
+------------------+-----------+
4 rows in set (0.00 sec)

# 查看主节点的 Server_id
mysql> SHOW GLOBAL VARIABLES LIKE '%server%';
+---------------------------------+--------------------------------------+
| Variable_name                   | Value                                |
+---------------------------------+--------------------------------------+
| character_set_server            | utf8mb4                              |
| collation_server                | utf8mb4_general_ci                   |
| innodb_ft_server_stopword_table |                                      |
| server_id                       | 1                                    |
| server_id_bits                  | 32                                   |
| server_uuid                     | 31a34ca9-bbc5-11ec-923c-000c29c2e995 |
+---------------------------------+--------------------------------------+
6 rows in set (0.00 sec)
```

4）在 Master 节点上创建复制权限的用户

**授权用户 user 从哪台服务器 host 能够登录：主节点上创建用户–>允许 188.188 网段的 IP，通过 slave 访问主节点**

```sql
mysql> CREATE USER 'slave'@'188.188.%.%' IDENTIFIED BY 'slavepassword';
Query OK, 0 rows affected (0.00 sec)

mysql> grant replication slave on *.* to 'slave'@'188.188.%.%' identified by 'slavepassword';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

**参数说明**

1. `*.*` 指定能操作所有数据库和表
2. `slave` 创建的用户名
3. `188.188.%.%` 来源 ID，即从库 IP(可指定 IP)
4. `slavepassword` 用户密码

<font color=red>注意：如条件允许，为了数据同步一致，建议先锁定一下表</font>

```sql
mysql> flush tables with read lock;
Query OK, 0 rows affected (0.00 sec)

mysql> show master status;
+------------------+----------+--------------+--------------------------------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB                                 | Executed_Gtid_Set |
+------------------+----------+--------------+--------------------------------------------------+-------------------+
| mysql-bin.000004 |      856 | repldb       | mysql,information_schema,performation_schema,sys |                   |
+------------------+----------+--------------+--------------------------------------------------+-------------------+
1 row in set (0.00 sec)
```

- File：当前记录 `bin-log` 的文件
- Position：从库读取的位置
- Binlog_Do_DB：需要同步的数据库
- Binlog_Ignore_DB：忽略的数据库，不同步

<font color=red>**查看 Master 主库状态，记录二进制文件名 (File) 和 位置 (Position)**</font>。至此，主库配置完成！

5）备份主库，并传到从库

```bash
$ mysqldump -uroot -p -A > /tmp/all.sql
$ ls /tmp/
all.sql
$ scp /tmp/all.sql root@188.188.4.211:/tmp
```

### Slave 配置

1）编辑从节点数据库配置文件

```bash
$ vim /etc/my.cnf
[mysqld]
lower_case_table_names=1               # 表名不区分大小写
server-id=2                            # [必须]服务器唯一ID,默认是1，多主从时注意不可重复
#log-bin=mysql-bin                     # 开启二进制日志(从节点如后面无级联的从节点，可以不开，避免无谓资源消耗)
relay-log=mysql-relay-log              # 打开从服务器中继日志文件
relay-log-index=mysql-relay-log.index  # 打开从服务器中继日志文件索引
```

2）导入数据库数据，重启加载配置

```bash
$ mysql -uroot -p < /tmp/all.sql
$ systemctl restart mysql && systemctl status mysql
```

3）在从库进行，执行同步 SQL 语句

```sql
mysql> CHANGE MASTER TO 
    -> MASTER_HOST='188.188.4.210',           # 主库IP地址
    -> MASTER_PORT=3306,                      # 主库端口                   
    -> MASTER_USER='slave',                   # 主库用于复制的用户    
    -> MASTER_PASSWORD='slavepassword',       # 密码
    -> MASTER_LOG_FILE='mysql-bin.000004',    # 主库日志名
    -> MASTER_LOG_POS=856;                    # 主库日志偏移量，即从何处开始复制
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> CHANGE MASTER TO MASTER_HOST='188.188.4.210',MASTER_PORT=3306,MASTER_USER='slave',MASTER_PASSWORD='slavepassword',MASTER_LOG_FILE='mysql-bin.000004',MASTER_LOG_POS=856;
```

4）从库启动 slave 线程，并检查

```sql
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 188.188.4.210
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 856
               Relay_Log_File: mysql-relay-log.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes           # Slave这两项必须为Yes
            Slave_SQL_Running: Yes

mysql> show processlist\G;
*************************** 1. row ***************************
     Id: 2
   User: root
   Host: localhost
     db: NULL
Command: Query
   Time: 0
  State: starting
   Info: show processlist
*************************** 2. row ***************************
     Id: 3
   User: system user
   Host: 
     db: NULL
Command: Connect
   Time: 44
  State: Waiting for master to send event
   Info: NULL
*************************** 3. row ***************************
     Id: 4
   User: system user
   Host: 
     db: NULL
Command: Connect
   Time: 44
  State: Slave has read all relay log; waiting for more updates
   Info: NULL
3 rows in set (0.00 sec)
```

如出现问题，就先停止 Slave，查看日志并根据信息解决后再开启 Slave

**注意：如果前面对 【主库】做了锁表操作，此时需要： 【 对 Master 解除 table（表）的锁定： "unlock tables;" 】**
