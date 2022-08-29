# NextCloud Install(LNMP)

> 文中的环境可能因人而异，所以如按教程搭建不成功的话，请注意查看 NextCloud、Nginx 等相关报错信息和日志记录，以下内容仅供参考；

## 环境配置

1）设置 Firewalld 和 SELinux

```bash
$ setenforce 0                                    
$ sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

$ firewall-cmd --permanent --add-service=http
$ firewall-cmd --permanent --add-service=https
$ firewall-cmd --permanent --zone=public --add-port=3306/tcp
$ firewall-cmd --reload
```

## Nginx

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

## MySQL

1）清除其他数据库信息

```bash
$ rpm -qa mysql
$ rpm -qa | grep mariadb
$ rpm -e --nodeps mariadb-libs
```

2）安装依赖环境并安装

```bash
$ yum -y install wget cmake gcc gcc-c++ ncurses numactl ncurses-devel libaio-devel openssl openssl-devel libaio
$ wget https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
$ tar -xf mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz -C /usr/local
$ mv /usr/local/mysql-8.0.20-linux-glibc2.12-x86_64 /usr/local/mysql
```

3）创建用户&组、目录

```bash
$ groupadd mysql
$ useradd -r -g mysql -s /sbin/nologin mysql
$ mkdir -p /usr/local/mysql/{data,etc,log}
$ chown -R mysql:mysql /usr/local/mysql/
$ cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
$ vim /etc/init.d/mysql
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
```

4）创建配置文件

```bash
$ vim /etc/my.cnf
[mysql]
port = 3306
socket = /usr/local/mysql/data/mysql.sock
default-character-set=utf8

[mysqld]
port = 3306
mysql_port = 33060
mysql_socket = /usr/local/mysql/data/mysql.sock
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
socket = /usr/local/mysql/data/mysql.sock
pid-file = /usr/local/mysql/data/mysqld.pid
log-error = /usr/local/mysql/log/error.log
#这个就是用之前的身份认证插件
default-authentication-plugin = mysql_native_password
#保证日志的时间正确
log_timestamps = SYSTEM
sort_buffer_size=10M
```

5）初始化数据库，并查看日志

```bash
$ cd /usr/local/mysql/bin/
$ ./mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
$ tailf /usr/local/mysql/log/error.log
2022-02-12T14:20:05.371035+08:00 0 [System] [MY-013169] [Server] /usr/local/mysql/bin/mysqld (mysqld 8.0.20) initializing of server in progress as process 1450
2022-02-12T14:20:05.380536+08:00 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2022-02-12T14:20:07.661113+08:00 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2022-02-12T14:20:09.814299+08:00 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: WlkhiG#.c0el
```

6）启动服务并修改密码、开启远程用户

```mysql
$ service mysql start
$ ./mysql -uroot -p
# 修改临时密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
# 创建远程用户
mysql> create user 'root'@'%' identified by 'password';
mysql> grant all privileges on *.* to 'root'@'%' with grant option;
# 查看用户创建情况
mysql> use mysql;
mysql> select host, user, authentication_string, plugin from user;
mysql> flush privileges;
mysql> exit
```

7）设置开机自启动

```bash
$ chkconfig --add mysql
$ chkconfig mysql on
$ chkconfig --list

$ vim /etc/profile
# mysql
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin
$ source /etc/profile

$ systemctl restart mysql
```

## Php

1）下载依赖并编译安装

```bash
$ yum -y install libsmbclient-devel libsodium ImageMagick-devel gmp gmp-devel libicu libicu-devel libxslt libxslt-devel autoconf automake libtool bzip2 bzip2-devel libxml2-devel sqlite-devel libcurl-devel oniguruma-devel libpng-devel libjpeg-devel freetype-devel libzip-devel openssl-devel
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

## Redis

1）安装依赖环境

```bash
$ gcc -v 
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)

$ yum -y install centos-release-scl scl-utils-build
$ yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
$ scl enable devtoolset-9 bash

# 注意：scl命令启用只是临时的，退出xshell或者重启就会恢复到原来的gcc版本。如果要长期生效的话，执行如下：
$ echo "source /opt/rh/devtoolset-9/enable" >>/etc/profile
```

2）下载编译安装

```bash
$ wget https://download.redis.io/releases/redis-6.2.6.tar.gz
$ tar -xf redis-6.2.6.tar.gz && cd ./redis-6.2.6
$ make && make install PREFIX=/usr/local/redis
$ mkdir /usr/local/redis/etc && cp redis.conf /usr/local/redis/etc/
```

3）创建开机自启服务

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

## NextCloud

[Nextcloud 官方地址]:https://nextcloud.com/support/
[Nextcloud Github 文档]:https://github.com/nextcloud/documentation/tree/master/admin_manual/installation

1）下载程序

```bash
$ wget https://download.nextcloud.com/server/releases/nextcloud-23.0.1.zip
$ unzip nextcloud-23.0.1.zip
$ mv ./nextcloud/* /usr/local/nginx/html/
```

2）创建对应的用户和数据库

```mysql
$ mysql -u root -p
mysql> create database nextcloud_db;

mysql> create user yuen@localhost identified by 'password';
mysql> grant all privileges on nextcloud_db.* to yuen@localhost with grant option;

mysql> create user 'yuen'@'%' identified by 'password';
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
Email Address []:yuen@hotmail.com                                       # 邮箱

$ ll
total 8
-rw-r--r--. 1 root root 1399 Dec  4 09:48 nextcloud.crt
-rw-r--r--. 1 root root 1704 Dec  4 09:48 nextcloud.key

$ chmod 600 /usr/local/nginx/cert/*
```

4）添加配置文件

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
$ vim /usr/local/nginx/vhosts/nextcloud.conf
upstream php-handler {
    server 127.0.0.1:9000;
}

server {
    listen 80;
    listen [::]:80;
    server_name 127.0.0.1;

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

5）浏览器打开 https://188.188.3.4 (域名)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220214173001494.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220214173029053.png)

**最终效果：**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220407172105024.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220407172243529.png)

## 异常问题

1）php 编译失败

**下载安装 CMake**

> CMake Error at CMakeLists.txt:1 (cmake_minimum_required):
> CMake 3.0.2 or higher is required.  You are running version 2.8.12.2

卸载旧版本再下载新版本安装

```bash
$ yum remove cmake -y
$ wget https://cmake.org/files/v3.22/cmake-3.22.1.tar.gz
$ tar -xf cmake-3.22.1.tar.gz && cd ./cmake-3.22.1
$ ./bootstrap
$ gmake && gmake install
$ ln -s /usr/local/bin/cmake /usr/bin/
```

**下载安装 oniguruma**

> No package oniguruma-devel available.
> configure: error: Package requirements (oniguruma) were not met:
> No package 'oniguruma' found

- 方法一：

```bash
$ yum -y install http://mirror.centos.org/centos-7/7.9.2009/cloud/x86_64/openstack-queens/Packages/o/oniguruma-6.7.0-1.el7.x86_64.rpm
$ yum -y install http://mirror.centos.org/centos-7/7.9.2009/cloud/x86_64/openstack-queens/Packages/o/oniguruma-devel-6.7.0-1.el7.x86_64.rpm
```

- 方法二：到 [oniguruma Github 地址](https://github.com/kkos/oniguruma) 下载编译安装

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

**下载安装 libzip**

> checking for libzip >= 0.11 libzip != 1.3.1 libzip != 1.7.0... no
> configure: error: Package requirements (libzip >= 0.11 libzip != 1.3.1 libzip != 1.7.0) were not met:
>
> Requested 'libzip >= 0.11' but version of libzip is 0.10.1

卸载旧版本再下载新版本安装

- 方法一：

```bash
$ yum remove libzip -y
$ wget https://libzip.org/download/libzip-1.8.0.tar.gz
$ tar -xf libzip-1.8.0.tar.gz && cd ./libzip-1.8.0
$ mkdir build && cd build \
&& cmake -DCMAKE_INSTALL_PREFIX=/usr .. \
&& make \
&& make install
```

- 方法二：

```bash
$ yum remove libzip -y
$ rpm -Uvh http://rpms.remirepo.net/enterprise/remi-release-7.rpm 
$ yum —enablerepo=remi install libzip5-devel 
```

2）redis 启动报 WARNING

```bash
WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
解决方法：将/proc/sys/net/core/somaxconn值设置为redis配置文件中的tcp-baklog值一致即可
$ echo '511' > /proc/sys/net/core/somaxconn
# 上述为临时，下述为永久处理
$ echo 'net.core.somaxconn= 1024' >> /etc/sysctl.conf
$ sysctl -p

WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl...is to take effect.
原因分析：overcommit_memory设置为0，在内存不足的情况下，后台保存会失败，要解决这个问题需要将此值改为1，然后重新加载，使其生效
$ echo 'vm.overcommit_memory=1' >> /etc/sysctl.conf 
$ sysctl -p

WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command .
警告：您的内核中启用了透明的大页面（THP）支持。这将创建与ReDIS的延迟和内存使用问题。若要修复此问题，请运行命令“EngEng/mS/mL/mM/ExpListNo.HugPoIP/启用”为root，并将其添加到您的/etc/rc.local，以便在重新启动后保留设置。在禁用THP之后，必须重新启动redis。
$ echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled
# 上述为临时，下述为永久处理,将如下加入到/etc/rc.local
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
   echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi

Increased maximum number of open files to 10032 (it was originally set to 1024).

```

3）内部服务器错误

> 登录无限循环或提示服务器不能完成您的请求、内部服务器错误不能完成请求等，请求的 ID 会不断改变，查询 logo 也没有太多有用的信息；

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220408091203.png)

- 原因一：

**基本是 php session 权限的问题**，首先检查 php-fpm 的设置，确保 user 和 group 和 web 服务器一致

```bash
$ vim /usr/local/php/etc/php-fpm.d/www.conf
# 根据客户端替换
user = nginx
group = nginx

$ chown -R root:nginx /usr/local/php/include/php/ext/session/
$ chown nginx:nginx -R /usr/local/nginx/html
$ systemctl restart php-fpm

# 如没有作用或你不知道web服务器用户是什么，可以试下下述命令再清空浏览器Cookie，就可以登陆成功；
$ chmod -R 777 /usr/local/php/include/php/ext/session/
```

- 原因二：

**MySQL-sort_buffer_size 参数**

```bash
$ vim /etc/my.cnf
[mysqld]
# 根据实际分配，MySQL官方文档推荐范围为256KB~2MB
sort_buffer_size=10M
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220214172838493.png)



