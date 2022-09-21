# MySQL5 Install(COM)

> MySQL5.7.x 编译安装，5.7 以上版本安装都依赖 boost 库

## 版本环境

- System：CentOS7.9.2009 Minimal
- MySQL：mysql-5.7.34-linux-glibc2.12-x86_64

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

1）下载程序包并解压至指定目录

```bash
$ cd /usr/local/src/
$ wget https://cdn.mysql.com/archives/mysql-5.7/mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz
$ tar -zxvf mysql-5.7.35-linux-glibc2.12-x86_64.tar.gz -C /usr/local/ && cd /usr/local
$ mv mysql-5.7.35-linux-glibc2.12-x86_64  mysql && cd ./mysql/
```

2）添加 mysql 组和 mysql 用户

```bash
$ groupadd mysql
$ useradd -r -g mysql mysql
```

3）创建配置文件

```bash
$ vim /etc/my.cnf
[mysql]
# 设置mysql客户端默认字符集 
default-character-set=utf8
socket=/usr/local/mysql/mysql.sock
[mysqld]
#skip-name-resolve 
#设置3306端口 
port = 3306
socket=/usr/local/mysql/mysql.sock
# 设置mysql的安装目录 
basedir=/usr/local/mysql
# 设置mysql数据库的数据的存放目录 
datadir=/usr/local/mysql/data
# 允许最大连接数 
max_connections=200
# 服务端使用的字符集默认为8比特编码的latin1字符集 
character-set-server=utf8
# 创建新表时将使用的默认存储引擎 
default-storage-engine=INNODB
#lower_case_table_name=1 
max_allowed_packet=16M
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

4）修改当前目录拥有者，并初始化 mysqld

```bash
$ chown -R mysql:mysql ./
$ ./bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
# 执行完毕后会自动生成默认的密码在执行记录中，注意复制出来 =R,lFXr(Y4b5
```

5）设置开机自启

```bash
# 复制启动脚本到资源目录，并增加可执行权限
$ cp ./support-files/mysql.server /etc/rc.d/init.d/mysqld
$ chmod +x /etc/rc.d/init.d/mysqld

# 将 mysqld 服务加入到系统服务
$ chkconfig --add mysqld
# 检查 mysqld 服务是否已经生效
$ chkconfig --list mysqld
mysqld         	0:off	1:off	2:on	3:on	4:on	5:on	6:off

$ service mysqld start
$ systemctl status mysqld
● mysqld.service - LSB: start and stop MySQL
   Loaded: loaded (/etc/rc.d/init.d/mysqld; bad; vendor preset: disabled)
   Active: active (exited) since Wed 2021-08-04 17:35:16 CST; 7s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 4711 ExecStart=/etc/rc.d/init.d/mysqld start (code=exited, status=0/SUCCESS)

Aug 04 17:35:16 data systemd[1]: Starting LSB: start and stop MySQL...
Aug 04 17:35:16 data mysqld[4711]: Starting MySQL SUCCESS!
Aug 04 17:35:16 data systemd[1]: Started LSB: start and stop MySQL.
Aug 04 17:35:16 data mysqld[4711]: 2021-08-04T09:35:16.706754Z mysqld_safe A mysqld process already exists
```
查看 mysqld 服务已经生效，在2、3、4、5运行级别随系统启动而自动启动，以后可以使用 service 命令控制 mysql 的启动和停止

- 查看启动项：chkconfig --list | grep -i mysql
- 删除启动项：chkconfig --del mysql

6）环境变量配置

> 将 mysql 的 bin 目录加入 PATH 环境变量，编辑 /etc/profile 文件

```bash
$ vim /etc/profile
# MySQL
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin
$ source /etc/profile
```

## 配置 MySQL

> 修改密码并开启远程访问

1）测试登录并修改密码

```sql
$ mysql -u root -p
Enter password:输入默认的临时密码
> set password=password('新密码');
> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '访问密码';
> flush privileges;
```

2）开启远程登录

确认关闭或开启防火墙 mysql-3306 端口的外部访问

```bash
$ firewall-cmd --zone=public --add-port=3306/tcp --permanent
$ firewall-cmd --reload
```

- `--zone`: 作用域，网络区域定义了网络连接的可信等级
- `--add-port`: 添加端口与通信协议，格式为：端口/通讯协议，协议是tcp 或 udp
- `--permanent`: 永久生效，没有此参数系统重启后端口访问失效

使用 root 需要赋予远程权限

```sql
mysql> GRANT ALL ON *.* TO user@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
mysql> flush privileges;
```