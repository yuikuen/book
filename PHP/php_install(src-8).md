# PHP Install(SRC-8)

> CentOS7.x 部署 Php8.x

1）安装依赖工具(一般使用下述命令即可，如不成功请按提示升级相关工具库，否则会提示编译失败)

```bash
$ yum -y install libtool automake libzip-devel epel-release libxml2 libxml2-devel openssl openssl-devel curl-devel libjpeg-devel libpng-devel freetype-devel libmcrypt-devel uuid libuuid-devel gcc bzip2 bzip2-devel gmp-devel readline-devel libxslt-devel autoconf bison gcc gcc-c++ sqlite-devel cmake
```

**下载安装 CMake**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220212160831028.png)

> CMake Error at CMakeLists.txt:1 (cmake_minimum_required):
>   CMake 3.0.2 or higher is required.  You are running version 2.8.12.2

卸载旧版本再下载新版本安装

```bash
$ yum remove cmake -y
$ wget https://cmake.org/files/v3.22/cmake-3.22.1.tar.gz
$ tar -zxvf cmake-3.22.1.tar.gz && cd cmake-3.22.1
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

- 方法二：(方法一下载较慢时可选择编译安装)

[oniguruma Github 地址]:https://github.com/kkos/oniguruma

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220212154256608.png)

```bash
$ wget https://github.com/kkos/oniguruma/archive/refs/tags/v6.9.7.1.tar.gz
$ tar -xf v6.9.7.1.tar.gz
$ cd oniguruma-6.9.7.1
$ ./autogen.sh
$ ./configure \
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

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220212160445815.png)

```bash
$ yum remove libzip -y
$ wget https://libzip.org/download/libzip-1.8.0.tar.gz
$ tar -zxvf libzip-1.8.0.tar.gz && cd libzip-1.8.0
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

2）下载安装 Php8.x

```bash
$ wget https://www.php.net/distributions/php-8.1.2.tar.gz
$ tar -xf php-8.1.2.tar.gz
$ cd php-8.1.2
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
--disable-short-tags

$ make && make install
```

3）创建配置文件

- 复制相关配置文件到 php 安装路径

```bash
$ cp /usr/local/src/php-8.1.2/sapi/fpm/init.d.php-fpm /usr/local/php/
$ cp /usr/local/src/php-8.1.2/php.ini-production /usr/local/php/etc/php.ini
```

- 创建 php-fpm.conf 配置文件

```bash
$ cd /usr/local/php/etc
$ cp php-fpm.conf.default php-fpm.conf && ll
-rw-r--r-- 1 root root  5376 Feb 12 17:04 php-fpm.conf
-rw-r--r-- 1 root root  5376 Feb 12 16:54 php-fpm.conf.default
drwxr-xr-x 2 root root    30 Feb 12 16:54 php-fpm.d
-rw-r--r-- 1 root root 72908 Feb 12 17:04 php.ini

$ cd /usr/local/php/etc/php-fpm.d
$ cp www.conf.default www.conf && ll
-rw-r--r-- 1 root root 20886 Feb 12 17:05 www.conf
-rw-r--r-- 1 root root 20886 Feb 12 16:54 www.conf.default
```

- 启动 php-fpm 测试启动脚本

```bash
$ cd /usr/local/php
$ bash init.d.php-fpm start
Starting php-fpm  done
```

4）创建开机自启服务

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
```

因之前已启动过 php-fpm，现 9000 端口被占用无法启动服务，杀死进程再启动服务

```bash
$ netstat -lntup | grep 9000
tcp        0      0 127.0.0.1:9000          0.0.0.0:*               LISTEN      29921/php-fpm: mast

$ killall php-fpm
$ systemctl enable --now php-fpm.service
$ systemctl status php-fpm
● php-fpm.service - php-fpm
   Loaded: loaded (/etc/systemd/system/php-fpm.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-02-12 17:08:09 CST; 5s ago
  Process: 29998 ExecStart=/usr/local/php/sbin/php-fpm (code=exited, status=0/SUCCESS)
 Main PID: 29999 (php-fpm)
   CGroup: /system.slice/php-fpm.service
           ├─29999 php-fpm: master process (/usr/local/php/etc/php-fpm.conf)
           ├─30000 php-fpm: pool www
           └─30001 php-fpm: pool www

Feb 12 17:08:09 nextcloud systemd[1]: Starting php-fpm...
Feb 12 17:08:09 nextcloud systemd[1]: Started php-fpm.
```

5）配置环境变量

```bash
$ vim /etc/profile
# php
export PHP_HOME=/usr/local/php
export PATH=$PATH:$PHP_HOME/bin
$ source /etc/profile

# 检查php-fpm是否正常安装
$ php -v
PHP 8.1.2 (cli) (built: Feb 12 2022 16:54:09) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.1.2, Copyright (c) Zend Technologies
```