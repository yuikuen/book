# MySQL8 Install(COM)

> MySQL8.x 编译安装，5.7 以上版本安装都依赖 boost 库

## 版本环境

- System：CentOS7.9.2009 Minimal
- MySQL：mysql-8.0.20-linux-glibc2.12-x86_64

1）卸载默认的 `mariadb`

```bash
$ rm -rf /etc/my.cnf /etc/my.cnf.d/
$ rpm -qa | grep mariadb | xargs yum remove -y
```


2）安装相关依赖包

```bash
$ yum -y install wget cmake gcc gcc-c++ ncurses numactl ncurses-devel libaio-devel openssl openssl-devel libaio
```

## 安装 MySQL

1）删除其它库信息

```bash
$ rpm -qa mysql
$ rpm -qa | grep mariadb
$ rpm -e --nodeps mariadb-libs
```

2）解压并改名

```bash
$ tar -xf mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
$ mv mysql-8.0.20-linux-glibc2.12-x86_64 /usr/local/mysql
```

3）创建用户&组、目录

```bash
$ groupadd mysql
$ useradd -r -g mysql -s /sbin/nologin mysql
$ mkdir /usr/local/mysql/{data,etc,log}
$ chown -R mysql:mysql /usr/local/mysql/
```

4）编辑配置文件

```bash
$ vim /usr/local/mysql/etc/my.cnf
[mysql]
port = 3306
socket = /usr/local/mysql/data/mysql.sock
[mysqld]
port = 3306
mysqlx_port = 33060
mysqlx_socket = /usr/local/mysql/data/mysqlx.sock
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
socket = /usr/local/mysql/data/mysql.sock
pid-file = /usr/local/mysql/data/mysqld.pid
log-error = /usr/local/mysql/log/error.log
#这个就是用之前的身份认证插件
default-authentication-plugin = mysql_native_password
#保证日志的时间正确
log_timestamps = SYSTEM
```

5）初始化数据库，并查看日志

```bash
$ cd /usr/local/mysql
$ bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
$ tailf /usr/local/mysql/log/error.log
2022-02-12T14:20:05.371035+08:00 0 [System] [MY-013169] [Server] /usr/local/mysql/bin/mysqld (mysqld 8.0.20) initializing of server in progress as process 1450
2022-02-12T14:20:05.380536+08:00 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2022-02-12T14:20:07.661113+08:00 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2022-02-12T14:20:09.814299+08:00 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: :p7mmIHz>4>b
```

注：记住临时密码，后面登录使用

6）设置启动文件，配置环境变量

```bash
$ cp /usr/local/mysql/support-files/mysql.server  /etc/init.d/mysqld
$ /etc/init.d/mysqld start
Starting MySQL.. SUCCESS!

$ vim /etc/profile
# mysql
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin
$ source /etc/profile
```

7）登录并配置远程登录 

```sql
$ mysql -u root -p
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
# 重置临时密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'Sinath90?';
Query OK, 0 rows affected (0.01 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select host, user, authentication_string, plugin from user;
+-----------+------------------+------------------------------------------------------------------------+-----------------------+
| host      | user             | authentication_string                                                  | plugin                |
+-----------+------------------+------------------------------------------------------------------------+-----------------------+
| localhost | mysql.infoschema | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED | caching_sha2_password |
| localhost | mysql.session    | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED | caching_sha2_password |
| localhost | mysql.sys        | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED | caching_sha2_password |
| localhost | root             | *9BFCDD1F1B1049D70D3BBE8EFD6684DC256EAAC9                              | mysql_native_password |
+-----------+------------------+------------------------------------------------------------------------+-----------------------+
4 rows in set (0.00 sec)
#这里就可以看到root@localhost这里的密码已经是mysql_native_password方式了
#创建远程用户登录
mysql> create user 'root'@'%' identified by 'Sinath90?';
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on *.* to 'root'@'%' with grant option;
Query OK, 0 rows affected (0.01 sec)

mysql> select host, user, authentication_string, plugin from user;
+-----------+------------------+------------------------------------------------------------------------+-----------------------+
| host      | user             | authentication_string                                                  | plugin                |
+-----------+------------------+------------------------------------------------------------------------+-----------------------+
| %         | root             | *9BFCDD1F1B1049D70D3BBE8EFD6684DC256EAAC9                              | mysql_native_password |
| localhost | mysql.infoschema | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED | caching_sha2_password |
| localhost | mysql.session    | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED | caching_sha2_password |
| localhost | mysql.sys        | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED | caching_sha2_password |
| localhost | root             | *9BFCDD1F1B1049D70D3BBE8EFD6684DC256EAAC9                              | mysql_native_password |
+-----------+------------------+------------------------------------------------------------------------+-----------------------+
5 rows in set (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye
```

8）配置开机自启动

```bash
$ cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
$ vim /etc/init.d/mysql
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data

$ systemctl status mysql.service
$ systemctl restart mysql.service
```