# PostgreSQL Install(YUM-12)

> CentOS7.x 部署 PostgreSQL12

## 版本环境

Jenkins 2.357 和将发布的 LTS 版本开始，Jenkins 需要 Java 11 才能使用，将放弃 Java 8，请注意兼容性问题。

- System：CentOS7.9.2009 Minimal
- PostgreSQL：postgresql-12

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 程序安装

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211030142958506.png)

1）安装存储库 RPM

```bash
$ sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

2）安装 [PostgreSQL](https://www.postgresql.org/download/linux/redhat/)

```bash
$ sudo yum install -y postgresql12-server
```

3）初始化数据库并设置开机自启

```bash
$ sudo /usr/pgsql-12/bin/postgresql-12-setup initdb
$ sudo systemctl enable --now postgresql-12
```

4）初始化用户&密码

```bash
# 切换用户，执行后提示符会变为'-bash-4.2$'
$ su - postgres
# 登录数据库，执行后提示符变为 'postgres=#'
psql -U postgres
# 设置密码，设置postgres用户密码为postgres
ALTER USER postgres WITH PASSWORD 'passwd';
# 退出数据库
\q
# 备注其他：列出所有库\l  列出所有用户\du 列出库下所有表\d
# 登出
exit
```

5）创建新用户和数据库

```bash
# 切换用户，执行后提示符会变为'-bash-4.2$'
su - postgres
# 登录数据库，执行后提示符变为 'postgres=#'
psql

# 创建用户和密码
create user user_name with password 'passwd';
# 创建数据库
create database db_test owner user_name encoding='UTF8';	//创建数据库指定所属者
# 将数据库得权限，全部赋给某个用户
grant all on database db_test to user_name;		//将dbtest所有权限赋值给username
```

6）修改配置文件，开通远程访问

```bash
$ cd /var/lib/pgsql/12/data
$ vim /var/lib/pgsql/12/data/postgresql.conf
# 修改如下内容：
#listen_addresses = 'localhost'  改为  listen_addresses='*'

$ vim /var/lib/pgsql/12/data/pg_hba.conf
# 修改如下内容，信任指定服务器连接
# 如果想允许所有IPv4地址，则加入一行host all all 0.0.0.0/0 md5。IPv6方法类似。
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
host    all             all             0.0.0.0/0               md5

# 重启postgresql服务
$ systemctl restart postgresql-12
```