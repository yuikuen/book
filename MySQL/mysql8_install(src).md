# MySQL8 Install(SRC)

> MySQL5.7.x 以上版本编译安装都依赖 boost 库，本文以 MySQL8.0.x 为例

## 版本环境

- System：CentOS7.9.2009 Minimal
- Cmake：cmake-3.24.1
- Gcc：gcc version 10.2.1
- Devtoolset：devtoolset-10
- MySQL：mysql-boost-8.0.28

卸载默认的 `mariadb`

```bash
$ rm -rf /etc/my.cnf /etc/my.cnf.d/
$ rpm -qa | grep mariadb | xargs yum remove -y
```

## 前置条件

安装依赖库 & 新版本 cmake、gcc、patchelf

```bash
$ yum -y install ncurses ncurses-devel gcc-* bzip2-* zlib* bison bison-devel openssl openssl-devel
```

### Cmake

> Centos7.x 默认版本过低，至少需 Cmake-3.5.x 以上，需到 [Cmake 官网](https://cmake.org/download/) 下载安装

```bash
$ yum -y remove cmake
$ wget https://github.com/Kitware/CMake/releases/download/v3.24.1/cmake-3.24.1.tar.gz
$ tar -xf cmake-3.24.1.tar.gz
$ cd cmake-3.24.1
$ ./bootstrap
$ gmake && gmake install
$ ln -s /usr/local/bin/cmake /usr/bin/

$ cmake -version
cmake version 3.24.1
CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

### Gcc

> MySQL8.0.x 要求 gcc 版本要 5.5 以上

MySQL8.0.x 对应 devtoolset-10，直接升级 devtoolset-10

```bash
$ yum -y install centos-release-scl
$ yum -y install devtoolset-10-gcc devtoolset-10-gcc-c++ devtoolset-10-binutils
$ scl enable devtoolset-10 bash
$ echo "source /opt/rh/devtoolset-10/enable" >>/etc/profile

$ gcc -v
gcc version 10.2.1 20210130 (Red Hat 10.2.1-11) (GCC)
```

### Patchelf

> CentOS7.x 的 Patchelf 依赖未下载成功，自行到 [NixOS/patchelf-GitHub](https://github.com/NixOS/patchelf/releases/) 下载安装

```bash
$ wget https://github.com/NixOS/patchelf/releases/download/0.15.0/patchelf-0.15.0.tar.gz
$ tar -xf patchelf-0.15.0.tar.gz
$ cd patchelf-0.15.0
$ ./configure
$ make && make install
```

## 安装 MySQL

1）创建用户和用户组

```bash
$ groupadd mysql
$ useradd mysql -g mysql -M -s /sbin/nologin
$ groups mysql
mysql : mysql
```

2）创建目录并下载程序

> 自行到 [官网](https://downloads.mysql.com/archives/) 下载并上传

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220827102549.png)

```bash
$ mkdir -p /opt/db/mysql/{data,logs,binlog,relaylog} /var/lib/mysql-files
$ cd /usr/local/src
$ wget https://cdn.mysql.com/archives/mysql-8.0/mysql-boost-8.0.28.tar.gz
```

3）解压编译安装

```bash
$ tar -xf mysql-boost-8.0.28.tar.gz
$ cd mysql-boost-8.0.28
$ cmake . \
-DCMAKE_INSTALL_PREFIX=/opt/db/mysql \
-DMYSQL_DATADIR=/opt/db/mysql/data \
-DMYSQL_UNIX_ADDR=/opt/db/mysql/mysql.sock \
-DENABLED_LOCAL_INFILE=1 \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DWITH_DEBUG=0 \
-DENABLE_DOWNLOADS=1 \
-DDOWNLOAD_BOOST=1 \
-DWITH_BOOST=boost \
-DFORCE_INSOURCE_BUILD=1 \
-DDOWNLOAD_BOOST_TIMEOUT=10000

# 确认内核数，开启多核加速编译
$ grep processor /proc/cpuinfo | wc -l
4
$ make -j 4 && make install
```

## 配置 MySQL

1）修改程序目录权限

```bash
$ chown -R mysql:mysql /var/lib/mysql-files /opt/db/mysql
$ chmod 750 /var/lib/mysql-files
```

2）配置环境变量

```bash
$ cat /etc/profile.d/mysql8.sh
export MYSQL_HOME=/opt/db/mysql
export PATH=$PATH:$MYSQL_HOME/bin

$ source /etc/profile
```

3）创建配置文件

```bash
$ vim /etc/my.cnf
[client]
default_character_set = utf8mb4
port    = 3306
socket  = /opt/db/mysql/mysql.sock

[mysqld]
character_set_server = utf8mb4
init_connect = 'SET NAMES utf8mb4'

user            = mysql
port            = 3306
socket          = /opt/db/mysql/mysql.sock
datadir         = /opt/db/mysql/data
log_error       = /opt/db/mysql/logs/mysql_error.log
log_bin         = /opt/db/mysql/binlog/binlog
relay_log       = /opt/db/mysql/relaylog/relaylog

max_connections = 10000
back_log = 3000
max_connect_errors = 7000

interactive_timeout = 60
wait_timeout = 60

auto_increment_increment = 3
auto_increment_offset = 1

max_allowed_packet = 32M
thread_cache_size = 300

replicate_wild_ignore_table = mysql.%
replicate_wild_ignore_table = test.%
replicate_wild_ignore_table = information_schema.%
replicate_wild_ignore_table = performance_schema.%

default_authentication_plugin=mysql_native_password
log_timestamps = SYSTEM
default_time_zone = '+8:00'
skip_external_locking = false
skip_name_resolve
explicit_defaults_for_timestamp=true
log_slave_updates = ON

slow_query_log = on
slow_query_log_file = /opt/db/mysql/logs/slow.log
long_query_time = 1

innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size=16M
innodb_buffer_pool_size=4096M
innodb_log_file_size=2048M
innodb_file_per_table = 1
innodb_open_files = 10240
innodb_flush_method=O_DIRECT

sync_binlog = 2
binlog_format = mixed
binlog_cache_size = 4M
max_binlog_size = 1G
binlog_expire_logs_seconds = 604800

server_id = 1057

[mysqldump]
quick
max_allowed_packet = 64M
```

4）初始化数据

```bash
$ cd /opt/db/mysql/bin
$ ./mysqld --initialize --user=mysql --basedir=/opt/db/mysql --datadir=/opt/db/mysql/data
```

5）配置启动服务

```bash
# 复制脚本文件
$ cp /opt/db/mysql/support-files/mysql.server /etc/init.d/
$ chmod u+x /etc/init.d/mysql.server

# 配置开机自启
$ chkconfig --add mysql.server
$ chkconfig mysql.server on

$ service mysql.server start
Starting MySQL... SUCCESS!

$ systemctl status mysql
● mysql.server.service - LSB: start and stop MySQL
   Loaded: loaded (/etc/rc.d/init.d/mysql.server; bad; vendor preset: disabled)
   Active: active (running) since Mon 2022-08-29 11:54:44 CST; 48s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 1129 ExecStart=/etc/rc.d/init.d/mysql.server start (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/mysql.server.service
           ├─1142 /bin/sh /opt/db/mysql/bin/mysqld_safe --datadir=/opt/db/mysql/data --pid-file=/opt/db/mysql/data/debug.pid
           └─2051 /opt/db/mysql/bin/mysqld --basedir=/opt/db/mysql --datadir=/opt/db/mysql/data --plugin-dir=/opt/db/mysql/lib/plugin --user=mysql --log-error=/opt/db/mysql/logs/mysql...

Aug 29 11:54:42 debug systemd[1]: Starting LSB: start and stop MySQL...
Aug 29 11:54:44 debug mysql.server[1129]: Starting MySQL.. SUCCESS!
Aug 29 11:54:44 debug systemd[1]: Started LSB: start and stop MySQL.
```

6）修改默认密码

```sql
$ egrep 'temporary|password' /opt/db/mysql/logs/mysql_error.log
2022-08-29T11:49:46.931754+08:00 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: 9sqj0p1S:Nuh

$ mysql -uroot -p
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码' PASSWORD EXPIRE NEVER;  
```

7）执行安全配置(可略)

> 生产环境建议执行

```bash
$ mysql_secure_installation
mysql_secure_installation: [ERROR] unknown variable 'default_character_set=utf8mb4'.

Securing the MySQL server deployment.
# 首先会提示用户密码验证，如空密码则进行创建密码
Enter password for user root: 

VALIDATE PASSWORD COMPONENT can be used to test passwords
and improve security. It checks the strength of password
and allows the users to set only those passwords which are
secure enough. Would you like to setup VALIDATE PASSWORD component?
# -->是否需要建立密码验证并修改root密码，选y则进入密码规则等创建过程
Press y|Y for Yes, any other key for No: 
Using existing password for root.
Change the password for root ? ((Press y|Y for Yes, any other key for No) : n

 ... skipping.
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.
# -->是否删除匿名用户
Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.


Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.
# -->是否禁止root远程登录
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y
Success.

By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.

# -->是否删除测试数据表
Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y
 - Dropping test database...
 ... Failed! Error: Commands out of sync; you can't run this command now
 - Removing privileges on test database...
 ... Failed! Error: Commands out of sync; you can't run this command now
Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
 ... Failed! Error: Commands out of sync; you can't run this command now
All done!
```

8）创建用户并授权

```sql
# 本地登录用户
mysql> create user 'test'@'localhost' identified by '123456';
# 远程登录用户
mysql> create user 'test'@'%' identified by '123456';

# 授权数据库所有表的所有操作给test用户
mysql> grant all privileges on *.* to 'test'@'localhost' with grant option;
mysql> grant all privileges on *.* to 'test'@'%' with grant option;
```