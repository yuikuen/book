# MySQL5.7 Install(SRC)

> MySQL5.7.x 以上版本编译安装都依赖 boost 库

## 版本环境

- System：CentOS7.9.2009 Minimal
- MySQL：mysql-boost-5.7.36

卸载默认的 `mariadb`

```bash
$ rm -rf /etc/my.cnf /etc/my.cnf.d/
$ rpm -qa | grep mariadb | xargs yum remove -y
```

## 安装说明

> 源码安装的教程可参考[官网](https://dev.mysql.com/doc/refman/5.7/en/source-installation.html) ，部署三步曲：配置-->编译-->安装

![image-20220401102821310](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/c0ba76101ab1f2a92413ab7f929c8356.png)

- 安装需求

| 安装需求            | 具体配置              |
| ------------------- | --------------------- |
| 安装目录（basedir） | /usr/local/mysql      |
| 数据目录（datadir） | /usr/local/mysql/data |
| 端口号              | 3306                  |
| socket文件位置      | $basedir/mysql.sock   |
| 字符集              | utf8mb4               |

- 常用配置选项

| 配置选项             | 描述                             | 默认值            | 建议值                                     |
| -------------------- | -------------------------------- | ----------------- | ------------------------------------------ |
| CMAKE_INSTALL_PREFIX | 安装基目录 (basedir)             | /usr/local/mysql  | 根据需求                                   |
| MYSQL_DATADIR        | 数据目录 (datadir)               | $basedir/data     | 根据需求                                   |
| SYSCONFDIR           | 默认配置文件 my.cnf 路径         |                   | /etc                                       |
| MYSQL_TCP_PORT       | TCP/IP 端口                      | 3306              | 非默认端口                                 |
| MYSQL_UNIX_ADDR      | 套接字 socket 文件路径           | /tmp/mysql.sock   | $basedir/                                  |
| DEFAULT_CHARSET      | 默认字符集                       | latin1            | **utf8mb4**                                |
| DEFAULT_COLLATION    | 默认校验规则                     | latin1_swedish_ci | utf8mb4_general_ci                         |
| WITH_EXTRA_CHARSETS  | 扩展字符集                       | all               | all                                        |
| ENABLED_LOCAL_INFILE | 是否启用本地加载外部数据文件功能 | OFF               | 建议开启                                   |
| WITH_SSL             | SSL 支持类型                     | system            | 建议显式指定                               |
| WITH_BOOST           | Boost 库源代码的位置             |                   | Boost 库是构建 MySQL 所必需的,建议事先下载 |

- 存储引擎相关配置项

以下选项值均为布尔值，0 或 1；0 代表不编译到服务器中，1 代表编译，建议都静态编译到服务器中。

其他的存储引擎可以根据实际需求在安装时通过 WITH_xxxx_STORAGE_ENGINE=1 的方式编译到服务器中。

| 配置选项                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| WITH_INNOBASE_STORAGE_ENGINE  | 将 InnoDB 存储引擎插件构建为静态模块编译到服务器中；建议编译到服务器中 |
| WITH_PARTITION_STORAGE_ENGINE | 是否支持分区                                                 |
| WITH_FEDERATED_STORAGE_ENGINE | 本地数据库是否可以访问远程 mysql 数据                        |
| WITH_BLACKHOLE_STORAGE_ENGINE | 黑洞存储引擎，接收数据，但不存储，直接丢弃                   |
| WITH_MYISAM_STORAGE_ENGINE    | 将 MYISAM 存储引擎静态编译到服务器中                         |

## 程序安装

1）删除旧版本的 MySQL 及相关配置文件，并安装相关依赖软件

```bash
$ rpm -qa mysql
$ rpm -qa | grep mariadb 
$ rpm -e --nodeps mariadb-libs 文件名
$ rm -rf /etc/my.cnf
```

2）下载依赖包&程序包，根据需求进行解压配置安装

```bash
$ yum -y install ncurses-devel cmake libaio-devel openssl-devel gcc gcc-c++ bison
$ tar -xf mysql-boost-5.7.36.tar.gz
$ cd mysql-5.7.37
```
注：环境不一样，可能还需要安装其他依赖包，比如 perl 相关的包

3）基于 cmake 进行配置

```bash
$ cmake . \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DMYSQL_DATADIR=/usr/local/mysql/data \
-DMYSQL_TCP_PORT=3306 \
-DMYSQL_UNIX_ADDR=/usr/local/mysql/mysql.sock \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DENABLED_LOCAL_INFILE=1 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8mb4 \
-DDEFAULT_COLLATION=utf8mb4_general_ci \
-DWITH_SSL=system \
-DWITH_BOOST=boost

选项说明：
-DCMAKE_INSTALL_PREFIX ：安装路径
-DMYSQL_DATADIR ：数据目录
-DMYSQL_TCP_PORT ：端口号
-DMYSQL_UNIX_ADDR ：套接字文件位置
```

4）开始编译安装

```bash
$ make -j2 && make install
选项说明：
-j2：代表同时开启多个线程共同实现编译操作
```

## 数据初始化

1）创建一个数据库专用账号 mysql(其所属组也为 mysql)

```bash
$ useradd -r -s /sbin/nologin mysql
$ id mysql
```

2）修改目录权限，并进入安装目录执行初始化操作

```bash
$ chown -R mysql:mysql /usr/local/mysql
$ cd /usr/local/mysql
# 注意最后输出的是默认密码，后期修改密码需用上
$ bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
...
2022-04-03T02:43:31.295939Z 1 [Note] A temporary password is generated for root@localhost: e!m#BqYjf9OG
```

3）拷贝 mysql.server 脚本到 `/etc/init.d` 目录，然后启动数据库

```bash
$ cp support-files/mysql.server /etc/init.d/mysql
$ service mysql start
Starting MySQL.Logging to '/usr/local/mysql/data/Dev-Pc.err'.
 SUCCESS!
```

4）编写 MySQL 配置文件

```bash
$ vim my.cnf
[mysqld]
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/usr/local/mysql/mysql.sock

[client]
default_character-set=utf8

$ service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

```bash
# 配置文件参考
$ cat /etc/my.cnf
[client]
port=3306
socket=/tmp/mysql.sock

[mysqld]
port = 3306
socket=/tmp/mysql.sock
user = mysql
server-id = 1
basedir = /opt/mysql
datadir = /opt/mysql/data
pid-file = /opt/mysql/data/mysqld.pid

default_storage_engine = InnoDB
max_allowed_packet = 512M
max_connections = 2048
open_files_limit = 65535
explicit_defaults_for_timestamp = 1

skip-name-resolve
lower_case_table_names=1

character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
init_connect='SET NAMES utf8mb4'

innodb_buffer_pool_size = 512M
innodb_log_file_size = 1024M
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit = 0

key_buffer_size = 64M

log-error = /opt/mysql/logs/error.log
log-bin = /opt/mysql/logs/binlog/mysql-bin

binlog_format = mixed
expire_logs_days = 10

slow_query_log = 1
slow_query_log_file = /opt/mysql/logs/slow.log
long_query_time = 1

[mysqldump]
quick
```

5）修改管理员密码

```bash
$ bin/mysqladmin -uroot password 'newpassword' -p
Enter password:e!m#BqYjf9OG
mysqladmin: [Warning] Using a password on the command line interface can be insecure.
Warning: Since password will be sent to server in plain text, use ssl connection to ensure password safety.
```

6）测试密码修改是否成功

```mysql
$ bin/mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.37 Source distribution

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

7）修改安全配置

```mysql
$ bin/mysql_secure_installation
# 第一项回车跳过外，其他都选y
Securing the MySQL server deployment.

Enter password for user root: 
The 'validate_password' plugin is installed on the server.
The subsequent steps will run with the existing configuration
of the plugin.
Using existing password for root.

Estimated strength of the password: 50 
Change the password for root ? ((Press y|Y for Yes, any other key for No) : 

 ... skipping.
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.
```

8）添加服务至开机启动项，并配置环境变量

```bash
$ chkconfig --add mysql
$ chkconfig mysql on

$ vim /etc/profile
# MySQL
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin
$ source /etc/profile
```

```bash
# 启动文件参考
$ cat /etc/systemd/system/mysqld.service
[Unit]
Description=MySQL Server
Documentation=man:mysqld(7)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql

Type=forking
PIDFile=/opt/mysql/data/mysqld.pid
TimeoutSec=0

ExecStart=/opt/mysql/bin/mysqld --defaults-file=/opt/mysql/etc/my.cnf --daemonize --pid-file=/opt/mysql/data/mysqld.pid $MYSQLD_OPTS 
EnvironmentFile=-/etc/sysconfig/mysql

LimitNOFILE = 5000
Restart=on-failure
RestartPreventExitStatus=1
PrivateTmp=false
```