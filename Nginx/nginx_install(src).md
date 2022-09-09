# Nginx Install(SRC)

> CentOS7.x 通用编译安装 [Nginx](http://nginx.org/)

## 版本环境

- System：CentOS7.9.2009 Minimal
- Nginx：nginx-1.21.4

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 

$ firewall-cmd --permanent --zone=public --add-port=80/tcp
$ firewall-cmd --reload
```

## 程序安装

1）下载安装依赖包

```bash
$ yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

- GCC 编译器
- PCRE 库-兼容正则表达式库，Rewrite 模块和 Http 核心模块都会使用到 PCRE 正则表达式
- Zlib 库-压缩算法，Nginx 的各个模块中需要使用 gzip 压缩
- Openssl 开放源代码软件库包-应用程序可以使用这个包进行安全通信，并且避免被窃听

2）下载源码并解压编译运行

```bash
$ wget http://nginx.org/download/nginx-1.21.4.tar.gz
$ tar -xf nginx-1.21.4.tar.gz && cd nginx-1.21.4
# 进入解压后的资源包，测试环境暂不指定任何参数，单一指定安装路径，直接执行编译并安装
$ ./configure --prefix=/usr/local/nginx 
$ make && make install 

# 配置环境变量
$ echo 'export PATH=/usr/local/nginx/sbin:$PATH' >> /etc/profile
$ source /etc/profile
$ nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```

确保系统防火墙关闭或端口已开放，浏览器输入本机 IP 地址，能看到 Nginx 欢迎页面就说明运行成功；

**Configure 可提供多个自定义安装配置参数，如：**[ `./configure –help` 查询详细参数]

> 在`./configure` 配置中，"--with" 表示启用模块，"--without" 表示禁用模块

```bash
# 参考例子，指定安装目录并且开启指定模块
$ ./configure --prefix=/usr/local/nginx --with-http_perl_module --with-http_stub_status_module --with-http_ssl_module --with-openssl-opt="enable-tlsext"
```

3）设置开机自启并查看状态

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

$ systemctl enable --now nginx && systemctl status nginx
● nginx.service
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2021-11-19 23:08:08 CST; 8ms ago
  Process: 21862 ExecStart=/usr/local/nginx/sbin/nginx (code=exited, status=0/SUCCESS)
 Main PID: 21863 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21863 nginx: master process /usr/local/nginx/sbin/nginx
           └─21865 nginx: worker process

Nov 19 23:08:08 share systemd[1]: Starting nginx.service...
Nov 19 23:08:08 share systemd[1]: Started nginx.service.

$ ps -ef | grep nginx
root     21863     1  0 23:08 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
nobody   21865 21863  0 23:08 ?        00:00:00 nginx: worker process
root     21867 16208  0 23:08 pts/0    00:00:00 grep --color=auto nginx
```