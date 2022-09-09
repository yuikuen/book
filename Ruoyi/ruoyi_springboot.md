# RuoYi SpringBoot

## 安装 Java

- JDK >= 1.8，自行到相关网站进行下载上传，并解压、配置环境变量

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
$ ln -s /usr/local/java/bin/java /usr/bin/java
```

## 安装 Maven 

- Maven >= 3.0，自行到相关网站进行下载上传，并解压、配置环境变量

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

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/c0ba76101ab1f2a92413ab7f929c8356.png)

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
$ chown -R mysql:mysql /usr/local/mysql

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

## 项目部署

1）从 git 仓库 clone 下来最新代码 [Ruoyi Gitee 地址](https://gitee.com/y_project/RuoYi)

```bash
$ cd /usr/local
$ git clone https://gitee.com/y_project/RuoYi.git
```

2）创建数据库并导入数据表

```mysql
mysql> create database ruoyi character set utf8 collate utf8_bin;
mysql> use ruoyi;
mysql> source /usr/local/RuoYi/sql/quartz.sql;
mysql> source /usr/local/RuoYi/sql/ry_20210924.sql;
mysql> show databases;
mysql> flush privileges;
```

3）修改配置文件 `application.yml & application-druid.yml` 的项目启动端口、数据源等

```yaml
$ vim RuoYi/ruoyi-admin/src/main/resources/application.yml
# 开发环境配置
server:
  # 服务器的HTTP端口，默认为80
  port: 10081
```

```yaml
$ vim RuoYi/ruoyi-admin/src/main/resources/application-druid.yml
# 数据源配置
spring:
    datasource:
        type: com.alibaba.druid.pool.DruidDataSource
        driverClassName: com.mysql.cj.jdbc.Driver
        druid:
            # 主库数据源
            master:
                # 数据库地址及库名注意要与之前创建的一致
                url: jdbc:mysql://localhost:3306/ruoyi?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
                username: root
                password: password
            # 从库数据源
            slave:
                # 从数据源开关/默认关闭
                enabled: false
                url:
                username:
                password:
```

4）打包并启动项目，`RuoYi/ruoyi-admin/target/ruoyi-admin.jar` 就是打包完成后的 jar 包文件

```bash
$ cd /usr/local/Ruoyi
$ mvn install
$ nohup java -jar /usr/local/RuoYi/ruoyi-admin/target/ruoyi-admin.jar &
```

```bash
$ cat nohup.out 
Application Version: 4.7.3
Spring Boot Version: 2.5.12
////////////////////////////////////////////////////////////////////
//                          _ooOoo_                               //
//                         o8888888o                              //
//                         88" . "88                              //
//                         (| ^_^ |)                              //
//                         O\  =  /O                              //
//                      ____/`---'\____                           //
//                    .'  \\|     |//  `.                         //
//                   /  \\|||  :  |||//  \                        //
//                  /  _||||| -:- |||||-  \                       //
//                  |   | \\\  -  /// |   |                       //
//                  | \_|  ''\---/''  |   |                       //
//                  \  .-\__  `-`  ___/-. /                       //
//                ___`. .'  /--.--\  `. . ___                     //
//              ."" '<  `.___\_<|>_/___.'  >'"".                  //
//            | | :  `- \`.;`\ _ /`;.`/ - ` : | |                 //
//            \  \ `-.   \_ __\ /__ _/   .-` /  /                 //
//      ========`-.____`-.___\_____/___.-`____.-'========         //
//                           `=---='                              //
//      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^        //
//             佛祖保佑       永不宕机      永无BUG               //
////////////////////////////////////////////////////////////////////
17:36:30.886 [background-preinit] INFO  o.h.v.i.util.Version - [<clinit>,21] - HV000001: Hibernate Validator 6.2.3.Final
17:36:30.897 [main] INFO  c.r.RuoYiApplication - [logStarting,55] - Starting RuoYiApplication using Java 1.8.0_331 on Dev-Pc with PID 2208 (/usr/local/RuoYi/ruoyi-admin/target/ruoyi-admin.jar started by root in /usr/local/RuoYi/ruoyi-admin/target)
17:36:30.897 [main] DEBUG c.r.RuoYiApplication - [logStarting,56] - Running with Spring Boot v2.5.12, Spring v5.3.18
17:36:30.898 [main] INFO  c.r.RuoYiApplication - [logStartupProfileInfo,681] - The following 1 profile is active: "druid"

$ ss -lntp|grep 18081
LISTEN     0      1000         *:18081                    *:*                   users:(("java",pid=2208,fd=36))
```
