# Nacos Install(SRC)

> CentOS7 部署单节点 Nacos

## 版本环境

- System：CentOS7.9.2009 Minimal
- Java：jdk-8u331-linux-x64
- Maven：apache-maven-3.8.5-bin
- MySQL：mysql-boost-5.7.37
- Nacos：nacos-server-2.0.3

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装 JDK

> 自行下载并上传服务器

```bash
$ tar -xf jdk-8u331-linux-x64.tar.gz -C /usr/local/
$ mv /usr/local/jdk1.8.0_331 /usr/local/java

$ vim /etc/profile
# java
JAVA_HOME=/usr/local/java
JRE_HOME=/usr/local/java/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH

$ source /etc/profile
```

## 安装 Maven

> 自行下载并上传服务器

```bash
$ tar -xf apache-maven-3.8.5-bin.tar.gz -C /usr/local/
$ mv /usr/local/apache-maven-3.8.5 /usr/local/maven

$ vim /etc/profile
# maven
export MAVEN_HOME=/usr/local/maven
export PATH=${JAVA_HOME}/bin:/usr/local/mysql/bin:${MAVEN_HOME}/bin:$PATH

$ source /etc/profile
```

修改 `Maven Setting.xml` 配置文件，配置参考 [阿里云云效 Maven](https://developer.aliyun.com/mvn/guide) 使用指南

```xml
$ mkdir -p /usr/local/maven/repo                            //创建本地仓库目录
$ vim /usr/local/maven/conf/settings.xml
   | Default: ${user.home}/.m2/repository
  -->
  <localRepository>/usr/local/maven/repo</localRepository>  //取消注释并修改本地仓库位置
  
# 添加阿里云公共仓库
<mirror>
  <id>aliyunmaven</id>
  <mirrorOf>*</mirrorOf>
  <name>阿里云公共仓库</name>
  <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

## 安装 MySQL

1）删除旧版本的 MySQL 及相关配置文件

```bash
$ rpm -qa mysql
$ rpm -qa | grep mariadb 
$ rpm -e --nodeps mariadb-libs  # 文件名
$ rm -rf /etc/my.cnf
```

2）安装相关依赖环境并下载源码包(自行下载上传)

![image-20220401102821310](https://cdn.jsdelivr.net/gh/yuikuen/picgo-cdn_images/img/c0ba76101ab1f2a92413ab7f929c8356.png)

```bash
$ yum -y install ncurses-devel cmake libaio-devel openssl-devel
```

3）解压目录并根据需求进行配置安装(基于 cmake 进行配置)

```bash
$ tar -xf mysql-boost-5.7.37.tar.gz
$ cd mysql-5.7.37

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

$ make -j2 && make install
```

4）数据初始化，创建一个数据库专用账号 mysql(其所属组也为 mysql)

```bash
$ useradd -r -s /sbin/nologin mysql
$ id mysql
$ cd /usr/local/mysql

# 创建mysql-files目录并修改权限
$ mkdir mysql-files
$ chown -R mysql:mysql /usr/local/mysql
$ chmod 750 mysql-files

# 数据库初始化操作，记录密码
$ bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
...
2022-04-03T02:43:31.295939Z 1 [Note] A temporary password is generated for root@localhost: ygnt#l2X(fzx
```

5）拷贝 mysql.server 脚本到 `/etc/init.d` 目录，编写 MySQL 配置文件，然后启动数据库

```bash
$ cp support-files/mysql.server /etc/init.d/mysql
$ service mysql start
Starting MySQL.Logging to '/usr/local/mysql/data/Dev-Pc.err'.
 SUCCESS!

$ vim my.cnf
[mysqld]
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/usr/local/mysql/mysql.sock

$ service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

6）重置管理员密码并设置安全配置

```mysql
$ bin/mysqladmin -uroot password 'newpassword' -p
Enter password:
mysqladmin: [Warning] Using a password on the command line interface can be insecure.
Warning: Since password will be sent to server in plain text, use ssl connection to ensure password safety.

# 重置后测试是否成功登录
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

mysql> exit

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

7）添加服务至开机启动并配置环境变量

```bash
$ chkconfig --add mysql
$ chkconfig mysql on

$ vim /etc/profile
# MySQL
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin

$ source /etc/profile
```

## 安装 Nacos

1）在 GitHub 上下载编译好的 nacos 安装包，下载地址: [Release](https://github.com/alibaba/nacos/releases)

```bash
$ tar -xvf nacos-server-2.0.3.tar.gz -C /usr/local/
```

2）登录 MySQL ，创建数据库并初始化 [SQL 数据源](https://github.com/alibaba/nacos/blob/master/distribution/conf/nacos-mysql.sql)

```mysql
mysql> create database nacos_db character set utf8 collate utf8_bin;
mysql> create user 'nacos'@'%' identified by 'nacos';
mysql> grant all privileges on nacos_db.* to 'nacos'@'%';
mysql> flush privileges;

# 切换数据库，执行nacos初始化数据脚本
mysql> use nacos_db;
mysql> source /usr/local/nacos/conf/nacos-mysql.sql;

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| nacos_db           |
| performance_schema |
| sys                |
+--------------------+
7 rows in set (0.00 sec)
```

3）配置 nacos 后端数据库，在 nacos 的 conf 目录下修改 [application.properties 配置文件](https://github.com/alibaba/nacos/blob/master/distribution/conf/application.properties)

```bash
$ vim /usr/local/nacos/conf/application.properties
#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
spring.datasource.platform=mysql

### Count of DB:
db.num=1

### Connect URL of DB:
# 注意：数据库地址、库名、时区的修改
db.url.0=jdbc:mysql://188.188.4.44:3306/nacos_db?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=Asia/Shanghai
db.user.0=nacos
db.password.0=nacos
```

4）脚本默认的启动方式为集群模式，需要修改成 standalone

```bash
$ vim ./nacos/bin/startup.sh
# 将55行的MODE改成standalone,单机启动
 54 export SERVER="nacos-server" 
 55 export MODE="standalone"
 56 export FUNCTION_MODE="all"
 57 export MEMBER_LIST=""
 58 export EMBEDDED_STORAGE=""
```

5）启动服务并测试

```bash
$ sh /usr/local/nacos/bin/startup.sh
/usr/local/java/bin/java -Djava.ext.dirs=/usr/local/java/jre/lib/ext:/usr/local/java/lib/ext  -Xms512m -Xmx512m -Xmn256m -Dnacos.standalone=true -Dnacos.member.list= -Xloggc:/usr/local/nacos/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/usr/local/nacos/plugins/health,/usr/local/nacos/plugins/cmdb -Dnacos.home=/usr/local/nacos -jar /usr/local/nacos/target/nacos-server.jar  --spring.config.additional-location=file:/usr/local/nacos/conf/ --logging.config=/usr/local/nacos/conf/nacos-logback.xml --server.max-http-header-size=524288
nacos is starting with standalone
nacos is starting，you can check the /usr/local/nacos/logs/start.out

$ tail -f /usr/local/nacos/logs/start.out
2022-04-27 14:43:22,299 INFO Creating filter chain: any request, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@3a08078c, org.springframework.security.web.context.SecurityContextPersistenceFilter@2b289ac9, org.springframework.security.web.header.HeaderWriterFilter@31ceba99, org.springframework.security.web.csrf.CsrfFilter@859ea42, org.springframework.security.web.authentication.logout.LogoutFilter@4487c0c2, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@73d3e555, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@18460128, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@53830483, org.springframework.security.web.session.SessionManagementFilter@bbf9e07, org.springframework.security.web.access.ExceptionTranslationFilter@2af46afd]

2022-04-27 14:43:22,443 INFO Initializing ExecutorService 'taskScheduler'

2022-04-27 14:43:22,464 INFO Exposing 16 endpoint(s) beneath base path '/actuator'

2022-04-27 14:43:22,593 INFO Tomcat started on port(s): 8848 (http) with context path '/nacos'

2022-04-27 14:43:22,597 INFO Nacos started successfully in stand alone mode. use external storage

2022-04-27 14:43:33,951 INFO Initializing Servlet 'dispatcherServlet'

2022-04-27 14:43:33,967 INFO Completed initialization in 15 ms
```

6）打开浏览器访问 nacos，默认账密 `nacos/nacos`

![image-20220427144340103](https://cdn.jsdelivr.net/gh/yuikuen/picgo-cdn_images/img/image-20220427144340103.png)

7）配置 systemd 管理 nacos

**注：**虽然系统已配置了 java 环境，并且直接调用 `startup.sh` 也能成功启动，但使用服务 `service` 就无法启动，原因是服务脚本的环境与系统环境变量不能共享

```bash
# 添加 nacos 服务运行用户
$ useradd -s /sbin/nologin -M nacos

# 修改 nacos 目录权限
$ chown -R nacos:nacos /usr/local/nacos

# 修改配置文件，注释并增加实际java路径
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/local/java
#[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
#[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
#[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/opt/taobao/java
#[ ! -e "$JAVA_HOME/bin/java" ] && unset JAVA_HOME

# 创建 service 文件
$ cat > /usr/lib/systemd/system/nacos.service <<EOF
[Unit]
Description=nacos
After=network.target

[Service]
Type=forking
Environment="JAVA_HOME=/usr/local/java"
ExecStart=/usr/local/nacos/bin/startup.sh
ExecReload=/usr/local/nacos/bin/shutdown.sh
ExecStop=/usr/local/nacos/bin/shutdown.sh
PrivateTmp=true
User=nacos
Group=nacos

[Install]
WantedBy=multi-user.target
EOF

$ systemctl daemon-reload && systemctl enable --now nacos.service && systemctl status nacos
```