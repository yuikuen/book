# Redis6.2源码安装

> Redis6 以上要求 gcc 版本必须是 5 以上，而 CentOS7.x 的 gcc 版本默认为 4.8.3。Red Hat 为了软件的稳定和版本支持，yum 上版本最新也是 4.8.3，所以无法使用 yum 进行软件更新，因此需要使用 Scl。
> Scl 软件集（Software Collections）是为了给 RHEL/CentOS 提供一种以方便、安全地安装和使用应用程序和运行时环境的多个版本方式，同时避免把系统搞乱

[Redis 官网地址]:https://redis.io/
[Redis 下载地址]:https://download.redis.io/releases/

![image-20220212172736237](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220212172736237.png)

1)安装依赖，升级 gcc 版本

```bash
$ gcc -v 
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)

$ yum -y install centos-release-scl scl-utils-build
$ yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
$ scl enable devtoolset-9 bash

# 注意：scl命令启用只是临时的，退出xshell或者重启就会恢复到原来的gcc版本。如果要长期生效的话，执行如下：
$ echo "source /opt/rh/devtoolset-9/enable" >>/etc/profile
$ source /etc/profile
```

2)下载 Reids 程序包并编译安装

```bash
$ wget https://download.redis.io/releases/redis-6.2.6.tar.gz
$ tar -xf redis-6.2.6.tar.gz
$ cd redis-6.2.6
$ make && make install PREFIX=/usr/local/redis
$ mkdir -p /usr/local/redis/etc /usr/local/redis/log && cp redis.conf /usr/local/redis/etc/
```

3)创建开机自启服务

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

$ systemctl daemon-reload && systemctl enable --now redis
$ systemctl status redis
● redis.service - Redis
   Loaded: loaded (/usr/lib/systemd/system/redis.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-02-12 17:59:49 CST; 28s ago
 Main PID: 1107 (redis-server)
   CGroup: /system.slice/redis.service
           └─1107 /usr/local/redis/bin/redis-server 127.0.0.1:6379
```

4)创建命令软件链接

```bash
$ ln -s /usr/local/redis/bin/redis-cli /usr/bin/redis
$ echo 'export PATH=/usr/local/redis/bin:$PATH' >> /etc/profile
$ source /etc/profile
```