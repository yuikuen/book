# NextCloud Install(VPS)

> 因之前部署的私有网盘总报磁盘错误，可能是虚拟挂载的问题。为保数据安全再次折腾一番，将硬盘做成直通方式，也方便后期迁移数据。
> 
> PS：个人娱乐，纯粹折腾，仅供参考！

## 环境配置

- System：CentOS7.9.2009 Minimal
- Nginx：nginx-1.22.0
- MySQL：mysql-boost-8.0.28
- Php：php-7.4.27
- Redis：redis-6.2.6
- NextCloud：nextcloud-24.0.4

下述磁盘操作如无需要，可跳过直接安装 `NMP + NextCloud`

**配置 Esxi 硬盘 RDM 直通**

> 通过 RDM（Raw Device Mapping）方式，将磁盘应设为本地 VMDK

1）通过命令查看当前挂载的所有磁盘信息

```bash
$ ls -l /dev/disks
...
t10.ATA_____ST2000DM0082D2FR102__________________________________WFL0JJVB
```

注：如磁盘过多，不好确认可用以下方法，找到设备的 `路径` 信息

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903151218.png)

2）通过命令进行挂载(名称可自定义)

```bash
$ vmkfstools -z /vmfs/devices/disks/<直通硬盘的标识符> /vmfs/volumes/<保存vmdk的硬盘标识符>/<VMDK名字>.vmdk

$ cd /vmfs/volumes
$ ls
60cb168f-40301f18-5d4d-2005250000bc  60dde79a-0ab3e169-fbf6-2005250000bc  61b21063-2edc3b9a-f208-2005250000bc  62163c1e-5435923f-f269-2005250000bc  Esxi-sys                             Test                                 f91a1177-3e2a97ca-74a3-8699935a1120
60cb1699-15579259-16ac-2005250000bc  60f990d3-f5624760-e1db-2005250000bc  620f70e6-8f083b7e-7601-2005250000bc  7ffa0a7a-a59448ba-5ea5-295f6f2a5ff4  SSD                                  Win                                  pcie
$ mkdir -p /vmfs/volumes/Esxi-sys/store.data
$ vmkfstools -z /vmfs/devices/disks/t10.ATA_____ST2000DM0082D2FR102__________________________________WFL0JJVB /vmfs/volumes/Esxi-sys/store.data/disk4.vmdk
```

3）创建虚拟机并添加磁盘

> 成功后会在指定路径下创建一个磁盘映射的 `*.vmdk` 文件

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903152512.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903152347.png)

4）挂载磁盘

> PS：因为我是安装好系统后才进行挂载操作，所以会有重启后磁盘错乱的问题，挂载后安装系统并不会有此问题

```bash
# 确认新磁盘 /dev/sda 的 UUID
$ sudo blkid -s UUID
/dev/sda: UUID="4a55017c-3639-4388-b794-f189ccfa6e76" 
/dev/sdb1: UUID="12271491-d64a-4032-8946-6edbdac0760a" 
/dev/sdb2: UUID="8V10ZR-Z2K0-pJps-7gpe-3eSf-3lQy-7c0YEC" 
/dev/sr0: UUID="2020-11-03-14-55-29-00" 
/dev/mapper/centos-root: UUID="b73b8b30-452c-4b7d-829e-304f2db46190" 
/dev/mapper/centos-swap: UUID="128253d9-f029-4a94-b6f9-4cbc96df309d"

$ sudo vim /etc/fstab
UUID=4a55017c-3639-4388-b794-f189ccfa6e76 /mnt/ex-storage         ext4    defaults        0 0
$ sudo mount -a
```

**安全配置：Firewalld 和 SELinux**

```bash
$ setenforce 0                                    
$ sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

$ firewall-cmd --permanent --add-service=http
$ firewall-cmd --permanent --add-service=https
$ firewall-cmd --permanent --zone=public --add-port=3306/tcp
$ firewall-cmd --reload
```

## 安装 NMP

### Nginx

1）安装依赖 & 库

```bash
$ yum -y install gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

2）解压编译安装

```bash
$ wget http://nginx.org/download/nginx-1.21.6.tar.gz
$ tar -xf nginx-1.21.6.tar.gz && cd ./nginx-1.21.6

$ ./configure \
--prefix=/usr/local/nginx \
--with-http_addition_module \
--with-http_stub_status_module \
--with-http_ssl_module \
--with-openssl-opt="enable-tlsext"

$ make && make install
```

3）创建权限用户

```bash
$ groupadd nginx
$ useradd -M -g nginx -s /sbin/nologin nginx
$ vim /usr/local/nginx/conf/nginx.conf
user nginx nginx;
```

4）设置开机自启动

```bash
$ vim /lib/systemd/system/nginx.service
[Unit]
Description=nginx
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target

$ systemctl enable --now nginx.service
```

### MySQL

1）卸载默认的 `mariadb`

```bash
$ rm -rf /etc/my.cnf /etc/my.cnf.d/
$ rpm -qa | grep mariadb | xargs yum remove -y
```

2）安装依赖库 & 新版本 cmake、gcc、patchelf

```bash
$ yum -y install ncurses ncurses-devel gcc-* bzip2-* zlib* bison bison-devel openssl openssl-devel
```

#### Cmake

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

#### Gcc

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

#### Patchelf

> CentOS7.x 的 Patchelf 依赖未下载成功，自行到 [NixOS/patchelf-GitHub](https://github.com/NixOS/patchelf/releases/) 下载安装

```bash
$ wget https://github.com/NixOS/patchelf/releases/download/0.15.0/patchelf-0.15.0.tar.gz
$ tar -xf patchelf-0.15.0.tar.gz
$ cd patchelf-0.15.0
$ ./configure
$ make && make install
```

#### 编译安装

1）创建用户和用户组

```bash
$ groupadd mysql
$ useradd mysql -g mysql -M -s /sbin/nologin
$ groups mysql
mysql : mysql
```

2）创建目录并下载程序

```bash
$ mkdir -p /opt/db/mysql/{data,logs,binlog,relaylog} /var/lib/mysql-files
$ cd /usr/local/src
$ wget https://cdn.mysql.com/archives/mysql-8.0/mysql-boost-8.0.28.tar.gz
```

3）解压编译安装

> !! 注意：此处有坑，源码编译的方式十分占用磁盘，建议分区大于 30G 以上 !!

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

#### 配置 MySQL

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

```m
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

### Php

1）下载依赖并编译安装

```bash
$ yum -y install libsmbclient-devel libsodium ImageMagick-devel gmp gmp-devel libicu libicu-devel libxslt libxslt-devel autoconf automake libtool bzip2 bzip2-devel libxml2-devel sqlite-devel libcurl-devel oniguruma-devel libpng-devel libjpeg-devel freetype-devel libzip-devel openssl-devel
```

上面的依赖可能无法安装到，会影响下面的操作，提前手工安装

- libzip

```bash
$ yum remove libzip -y
$ wget https://libzip.org/download/libzip-1.8.0.tar.gz
$ tar -xf libzip-1.8.0.tar.gz && cd ./libzip-1.8.0
$ mkdir build && cd build \
&& cmake -DCMAKE_INSTALL_PREFIX=/usr .. \
&& make \
&& make install
```

- oniguruma

```bash
$ wget https://github.com/kkos/oniguruma/archive/refs/tags/v6.9.7.1.tar.gz
$ tar -xf v6.9.7.1.tar.gz && cd ./oniguruma-6.9.7.1
$ ./autogen.sh && ./configure \
--bindir=/usr/sbin/ \
--sbindir=/usr/sbin/ \
--libexecdir=/usr/libexec \
--sysconfdir=/etc/ \
--localstatedir=/var \
--libdir=/usr/lib64/  \
--includedir=/usr/include/ \
--datarootdir=/usr/share \
--infodir=/usr/share/info \
--localedir=/usr/share/locale \
--mandir=/usr/share/man/ \
--docdir=/usr/share/doc/onig

$ make && make install
```

```bash
$ wget https://www.php.net/distributions/php-7.4.27.tar.gz
$ tar -xf php-7.4.27.tar.gz && cd ./php-7.4.27
$ ./configure \
--prefix=/usr/local/php/ \
--with-config-file-path=/usr/local/php/etc \
--with-config-file-scan-dir=/usr/local/php/etc/conf.d \
--enable-fpm \
--enable-soap \
--with-openssl \
--with-openssl-dir \
--with-zlib \
--with-iconv \
--with-bz2 \
--enable-gd \
--with-jpeg \
--with-freetype \
--with-curl \
--enable-dom \
--enable-xml \
--with-zip \
--enable-ftp \
--enable-mbstring \
--enable-pdo \
--with-pdo-mysql \
--with-zlib-dir \
--enable-session \
--enable-shmop \
--enable-simplexml \
--enable-sockets \
--enable-sysvmsg \
--enable-sysvsem \
--enable-sysvshm \
--with-xsl \
--enable-mysqlnd \
--with-mysqli \
--without-pear \
--enable-opcache \
--enable-fileinfo \
--disable-short-tags

$ make && make install
```

2）创建配置文件

```bash
$ cp ./sapi/fpm/init.d.php-fpm /usr/local/php/
$ cp ./php.ini-production /usr/local/php/etc/php.ini

$ cd /usr/local/php/etc/
$ cp php-fpm.conf.default php-fpm.conf
$ cp ./php-fpm.d/www.conf.default ./php-fpm.d/www.conf

$ cd /usr/local/php
$ bash init.d.php-fpm start
Starting php-fpm  done
```

3）创建开机自启

```bash
$ vim /etc/systemd/system/php-fpm.service
[Unit]
Description=php-fpm
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/php/sbin/php-fpm
PrivateTmp=True

[Install]
WantedBy=multi-user.target

# 因之前已启动过php-fpm，现9000端口被占用无法启动服务，杀死进程再启动服务
$ netstat -lntup | grep 9000
$ killall php-fpm
$ systemctl enable --now php-fpm.service
```

4）配置环境变量

```bash
$ vim /etc/profile
# php
export PHP_HOME=/usr/local/php
export PATH=$PATH:$PHP_HOME/bin
$ source /etc/profile

# 检查php-fpm是否正常安装
$ php -v
PHP 7.4.27 (cli) (built: Feb 14 2022 15:53:46) ( NTS )
Copyright (c) The PHP Group
Zend Engine v3.4.0, Copyright (c) Zend Technologies
```

### Redis

> 前面已升级过 gcc，此处直接部署即可，如有异常报错请参考 [Redis_Bug](../Redis/redis_bug.md)

1）下载编译安装

```bash
$ wget https://download.redis.io/releases/redis-6.2.6.tar.gz
$ tar -xf redis-6.2.6.tar.gz && cd ./redis-6.2.6
$ make && make install PREFIX=/usr/local/redis
$ mkdir /usr/local/redis/etc && cp redis.conf /usr/local/redis/etc/
```

2）创建开机自启服务

```bash
$ vim /usr/lib/systemd/system/redis.service
[Unit]
Description=Redis
After=network.target

[Service]
# Type=forking
PIDFile=/var/run/redis_6379.pid
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target

$ systemctl daemon-reload
$ systemctl enable --now redis

# 创建 redis 命令软链接
$ ln -s /usr/local/redis/bin/redis-cli /usr/bin/redis
```

## 安装 NextCloud

> 安装前可到 [官网](https://nextcloud.com/support/) 和 [GitHub](tps://github.com/nextcloud/documentation/tree/master/admin_manual/installation) 查看相关配置教程

1）下载程序并解压至指定目录

```bash
$ wget https://download.nextcloud.com/server/releases/nextcloud-24.0.4.zip
$ unzip nextcloud-24.0.4.zip
$ mv ./nextcloud/* /usr/local/nginx/html/
```

2）创建对应的用户和数据库

```sql
$ mysql -u root -p
mysql> create database cloud_db;

mysql> create user 'yuen'@'localhost' identified by 'password';
mysql> create user 'yuen'@'%' identified by 'password';

mysql> grant all privileges on *.* to 'yuen'@'localhost' with grant option;
mysql> grant all privileges on *.* to 'yuen'@'%' with grant option;
mysql> flush privileges;
mysql> exit
```

3）添加 ssl 证书

```bash
$ mkdir -p /usr/local/nginx/cert
$ openssl req -new -x509 -days 365 -nodes -out ./nextcloud.crt -keyout ./nextcloud.key
Generating a 2048 bit RSA private key
.....................................................................................................................+++
...................................................................................................+++
writing new private key to './nextcloud.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN                                    # 国家
State or Province Name (full name) []:GuangDong                         # 省份
Locality Name (eg, city) [Default City]:DG                              # 地区
Organization Name (eg, company) [Default Company Ltd]:CompanyName       # 公司名
Organizational Unit Name (eg, section) []:IT                            # 部门
Common Name (eg, your name or your server's hostname) []:Cloud          # 域名
Email Address []:yuikuen.yuen@hotmail.com                               # 邮箱

$ ll
total 8
-rw-r--r--. 1 root root 1399 Dec  4 09:48 nextcloud.crt
-rw-r--r--. 1 root root 1704 Dec  4 09:48 nextcloud.key

$ chmod 600 /usr/local/nginx/cert/*
```

4）修改 `nginx.conf` 配置文件，支持 php 模块

```bash
$ vim /usr/local/nginx/conf/nginx.conf
# 取消注释并开启支持php
location ~ \.php$ {
    root html;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include        fastcgi_params;
}

# 最后增加vhosts目录
include /usr/local/nginx/vhosts/*.conf;
```

```bash
$ mkdir -p /usr/local/nginx/vhosts
$ vim /usr/local/nginx/vhosts/cloud_data.conf
upstream php-handler {
    server 127.0.0.1:9000;
}

server {
    listen 80;
    listen [::]:80;
    server_name 188.188.4.254;

    # Enforce HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443      ssl;
    listen [::]:443 ssl;
    server_name 127.0.0.1;

    # Use Mozilla's guidelines for SSL/TLS settings
    # https://mozilla.github.io/server-side-tls/ssl-config-generator/
    ssl_certificate     /usr/local/nginx/cert/nextcloud.crt;
    ssl_certificate_key /usr/local/nginx/cert/nextcloud.key;

    # HSTS settings
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;

    # set max upload size and increase upload timeout:
    client_max_body_size 10240M;
    client_body_timeout 300s;
    fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Pagespeed is not supported by Nextcloud, so if your server is built
    # with the `ngx_pagespeed` module, uncomment this line to disable it.
    #pagespeed off;

    # HTTP response headers borrowed from Nextcloud `.htaccess`
    add_header Referrer-Policy                      "no-referrer"   always;
    add_header X-Content-Type-Options               "nosniff"       always;
    add_header X-Download-Options                   "noopen"        always;
    add_header X-Frame-Options                      "SAMEORIGIN"    always;
    add_header X-Permitted-Cross-Domain-Policies    "none"          always;
    add_header X-Robots-Tag                         "none"          always;
    add_header X-XSS-Protection                     "1; mode=block" always;

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Path to the root of your installation
    root /usr/local/nginx/html;

    # Specify how to handle directories -- specifying `/index.php$request_uri`
    # here as the fallback means that Nginx always exhibits the desired behaviour
    # when a client requests a path that corresponds to a directory that exists
    # on the server. In particular, if that directory contains an index.php file,
    # that file is correctly served; if it doesn't, then the request is passed to
    # the front-end controller. This consistent behaviour means that we don't need
    # to specify custom rules for certain paths (e.g. images and other assets,
    # `/updater`, `/ocm-provider`, `/ocs-provider`), and thus
    # `try_files $uri $uri/ /index.php$request_uri`
    # always provides the desired behaviour.
    index index.php index.html /index.php$request_uri;

    # Rule borrowed from `.htaccess` to handle Microsoft DAV clients
    location = / {
        if ( $http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/$is_args$args;
        }
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # Make a regex exception for `/.well-known` so that clients can still
    # access it despite the existence of the regex rule
    # `location ~ /(\.|autotest|...)` which would otherwise handle requests
    # for `/.well-known`.
    location ^~ /.well-known {
        # The rules in this block are an adaptation of the rules
        # in `.htaccess` that concern `/.well-known`.

        location = /.well-known/carddav { return 301 /remote.php/dav/; }
        location = /.well-known/caldav  { return 301 /remote.php/dav/; }

        location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
        location /.well-known/pki-validation    { try_files $uri $uri/ =404; }

        # Let Nextcloud's API for `/.well-known` URIs handle all other
        # requests by passing them to the front-end controller.
        return 301 /index.php$request_uri;
    }

    # Rules borrowed from `.htaccess` to hide certain paths from clients
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

    # Ensure this block, which passes PHP files to the PHP process, is above the blocks
    # which handle static assets (as seen below). If this block is not declared first,
    # then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
    # to the URI, resulting in a HTTP 500 error response.
    location ~ \.php(?:$|/) {
        # Required for legacy support
        rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+|.+\/richdocumentscode\/proxy) /index.php$request_uri;

        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;

        fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
        fastcgi_param front_controller_active true;     # Enable pretty urls
        fastcgi_pass php-handler;

        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;

        fastcgi_max_temp_file_size 0;
    }

    location ~ \.(?:css|js|svg|gif|png|jpg|ico|wasm|tflite)$ {
        try_files $uri /index.php$request_uri;
        expires 6M;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets

        location ~ \.wasm$ {
            default_type application/wasm;
        }
    }

    location ~ \.woff2?$ {
        try_files $uri /index.php$request_uri;
        expires 7d;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets
    }

    # Rule borrowed from `.htaccess`
    location /remote {
        return 301 /remote.php$request_uri;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
```

5）浏览器打开服务地址 https://188.188.4.254 (域名)

> 如出现 `内部服务器错误` **基本是 php session 权限的问题**，首先检查 php-fpm 的设置，确保 user 和 group 和 web 服务器一致

```bash
$ vim /usr/local/php/etc/php-fpm.d/www.conf
# 根据之前创建的 web 账户填写
user = nginx
group = nginx

$ chown -R root:nginx /usr/local/php/include/php/ext/session/
$ chown nginx:nginx -R /usr/local/nginx/html
$ systemctl restart php-fpm

# 如没有作用或你不知道web服务器用户是什么，可以试下下述命令再清空浏览器Cookie，就可以登陆成功；
$ chmod -R 777 /usr/local/php/include/php/ext/session/
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903162204.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903162359.png)

注：如有其他问题或异常，可参考 [NextCloud_Bug](nextcloud_bug.md)

## 反向代理

> 使用云上服务代理到本地私有云盘，实现 HTTPS 访问
> 
> PS：此处按个人需求，具体操作方法请自行找度娘

- 注册域名，并申请域名证书，如 `cloud.yuikuen.top`
- 购置云上服务器，并将域名解析至云上服务器
- 云上服务器安装 `Nginx` 服务，并上传域名证书，创建反向代理的配置文件

```bash
$ cat https_cloud.conf 
server {
    listen 80;
    listen 443 ssl;
    server_name cloud.yuikuen.top;

    ssl_certificate  /usr/local/nginx/cert/8359535_cloud.yuikuen.top.pem;
    ssl_certificate_key /usr/local/nginx/cert/8359535_cloud.yuikuen.top.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;

    client_max_body_size 10240M;
    client_body_timeout 300s;
    fastcgi_buffers 64 4K;

    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    add_header Referrer-Policy                      "no-referrer"   always;
    add_header X-Content-Type-Options               "nosniff"       always;
    add_header X-Download-Options                   "noopen"        always;
    add_header X-Frame-Options                      "SAMEORIGIN"    always;
    add_header X-Permitted-Cross-Domain-Policies    "none"          always;
    add_header X-Robots-Tag                         "none"          always;
    add_header X-XSS-Protection                     "1; mode=block" always;

    fastcgi_hide_header X-Powered-By;

    location / {
	proxy_pass https://183.xx.xx.xx/;
    }
}
```

**注：个人的云上服务器是国外的，无需作备案，并且访问效果并不佳，不建议各位折腾，我只是单纯图个乐而已，一般我是使用WireGuard来做透穿来使用**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903174506.png)
