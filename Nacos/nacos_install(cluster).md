# Nacos Install(Cluster)

> CentOS7 部署 Nacos 集群

## 版本环境

集群部署架构参照 [Nacos 官网](https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html)，在开源的时候把所有服务列表放到一个 vip 下，然后挂到一个域名下

- [http://ip1](http://ip1/):port/openAPI 直连 ip 模式，机器挂则需要修改 ip 才可以使用
- [http://SLB](http://slb/):port/openAPI 挂载 SLB 模式(内网 SLB，不可暴露到公网，以免带来安全风险)，直连 SLB 即可，下面挂 server 真实 ip，可读性不好
- [http://nacos.com](http://nacos.com/):port/openAPI 域名 + SLB 模式(内网 SLB，不可暴露到公网，以免带来安全风险)，可读性好，而且换 ip 方便，推荐模式

![img](https://nacos.io/img/deployDnsVipMode.jpg)

| 主机名    | IP 地址       | 角色                     | 服务                        |
| --------- | ------------- | ------------------------ | --------------------------- |
| ssd-dev01 | 188.188.4.210 | nacos,mysql-master,nginx | nacos,mysql,jdk,maven,nginx |
| ssd-dev02 | 188.188.4.211 | nacos,mysql-slave        | nacos,mysql,jdk,maven       |
| ssd-dev03 | 188.188.4.212 | nacos,mysql-slave        | nacos,mysql,jdk,maven       |

- System：CentOS7.9.2009 Minimal
- Java：jdk-8u291-linux-x64
- Maven：apache-maven-3.8.1-bin
- MySQL：mysql-boost-5.7.37
- Nacos：nacos-server-2.0.4
- Nginx：nginx-1.21.4

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

### 安装 JDK

```bash
$ wget https://download.oracle.com/otn/java/jdk/8u291-b10/d7fc238d0cbf4b0dac67be84580cfb4b/jdk-8u291-linux-x64.tar.gz
$ tar -xf jdk-8u291-linux-x64.tar.gz -C /usr/local/ 
$ mv /usr/local/jdk1.8.0_291 /usr/local/java

$ vim /etc/profile
# java
JAVA_HOME=/usr/local/java
JRE_HOME=/usr/local/java/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH

$ source /etc/profile
```

### 安装 Maven

```bash
$ wget https://mirrors.bfsu.edu.cn/apache/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
$ tar -xf apache-maven-3.8.1-bin.tar.gz -C /usr/local/ && mv /usr/local/apache-maven-3.8.1 /usr/local/maven

$ vim /etc/profile
# maven
export MAVEN_HOME=/usr/local/maven
export PATH=${JAVA_HOME}/bin:/usr/local/mysql/bin:${MAVEN_HOME}/bin:$PATH

$ source /etc/profile
```

**修改 Maven 的 Settings.xml**，配置参考 [阿里云云效 Maven](https://developer.aliyun.com/mvn/guide) 使用指南

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

### 安装 MySQL

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
2022-04-03T02:43:31.295939Z 1 [Note] A temporary password is generated for root@localhost: e!m#BqYjf9OG
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
Enter password:e!m#BqYjf9OG
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

### 配置主从

#### Master

1）编辑主节点数据库配置文件

```bash
$ vim /usr/local/mysql/my.cnf          # 一般默认为/etc/my.cnf，具体以实际环境为准
[mysqld]
lower_case_table_names=1               # 表名不区分大小写
binlog_format=MIXED                    # 复制模式，日志格式

server-id=1                            # [必须]服务器唯一ID,默认是1，多主从时注意不可重复
log-bin=mysql-bin                      # 开启二进制日志，可指定存储位置，默认mysql目录
log-bin-index=mysql-bin.index          # 打开二进制日志文件索引

//以下非必须项，仅供参考
sync_binlog=1                          # 控制数据库binlog刷到磁盘上，0不控制，性能最好，1每次事务提交都会刷到日志文件中，性能最差但最安全
expire_logs_days=7                     # 保留天数，以防日志占满磁盘，默认为0，表示不自动删除

#binlog-do-db=repldb                    # 需同步的二进制数据库名，可配置多个
binlog-ignore-db=mysql                 # 不同步的数据库，可配置多个
binlog-ignore-db=information_schema    
binlog-ignore-db=performation_schema
binlog-ignore-db=sys

auto-increment-offset=1                # 表中自增字段每次的偏移量
auto-increment-increment=1             # 表中自增字段每次的自增量
slave-skip-errors=all                  # 跳过从库错误
```

2）重启加载配置，查看是否正常生成 `log-bin` 文件

```bash
$ systemctl restart mysql && systemctl status mysql

$ mysql -uroot -p
# 查看二进制日志是否开启
mysql> show global variables like '%log%';
```

![image-20220415170520671](https://cdn.jsdelivr.net/gh/yuikuen/picgo-cdn_images/img/image-20220415170520671.png)

3）在 Master 节点上创建复制权限的用户 **主节点上创建用户–>允许 188.188 网段的 IP，通过 slave 访问主节点**

```mysql
mysql> CREATE USER 'slave'@'188.188.%.%' IDENTIFIED BY 'slavepassword';
Query OK, 0 rows affected (0.00 sec)

mysql> grant replication slave on *.* to 'slave'@'188.188.%.%' identified by 'slavepassword';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

<font color=red>注意：如条件允许，为了数据同步一致，建议先锁定一下表</font>

```mysql
mysql> flush tables with read lock;
Query OK, 0 rows affected (0.00 sec)

mysql> show master status;
+------------------+----------+--------------+--------------------------------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB                                 | Executed_Gtid_Set |
+------------------+----------+--------------+--------------------------------------------------+-------------------+
| mysql-bin.000001 |      856 |              | mysql,information_schema,performation_schema,sys |                   |
+------------------+----------+--------------+--------------------------------------------------+-------------------+
1 row in set (0.00 sec)
```

5）备份主库，并传到从库恢复数据，此为测试环境可选择性操作，生产环境按需操作

```bash
$ mysqldump -uroot -p -A > /tmp/all.sql
$ ls /tmp/
all.sql
$ scp /tmp/all.sql root@188.188.4.211:/tmp
```

#### Slave

> 一主多从，配置一样！注意 server-id 唯一即可；

1）编辑从节点数据库配置文件

- 188.188.4.211：server-id=2
- 188.188.4.212：server-id=3

```bash
$ vim /etc/my.cnf
[mysqld]
lower_case_table_names=1               # 表名不区分大小写
server-id=2                            # [必须]服务器唯一ID,默认是1，多主从时注意不可重复
#log-bin=mysql-bin                     # 开启二进制日志(从节点如后面无级联的从节点，可以不开，避免无谓资源消耗)
relay-log=mysql-relay-log              # 打开从服务器中继日志文件
relay-log-index=mysql-relay-log.index  # 打开从服务器中继日志文件索引
```

2）导入数据库数据，重启加载配置(数据导入按需操作)

```bash
$ mysql -uroot -p < /tmp/all.sql
$ systemctl restart mysql && systemctl status mysql
```

3）在从库进行，执行同步 SQL 语句

```mysql
mysql> CHANGE MASTER TO 
    -> MASTER_HOST='188.188.4.210',           # 主库IP地址
    -> MASTER_PORT=3306,                      # 主库端口                   
    -> MASTER_USER='slave',                   # 主库用于复制的用户    
    -> MASTER_PASSWORD='slavepassword',       # 密码
    -> MASTER_LOG_FILE='mysql-bin.000001',    # 主库日志名
    -> MASTER_LOG_POS=856;                    # 主库日志偏移量，即从何处开始复制
Query OK, 0 rows affected, 2 warnings (0.01 sec)

# 操作方法一样，任选其一
mysql> CHANGE MASTER TO MASTER_HOST='188.188.4.210',MASTER_PORT=3306,MASTER_USER='slave',MASTER_PASSWORD='slavepassword',MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=856;
```

4）从库启动 slave 线程，并检查

```mysql
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 188.188.4.210
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 856
               Relay_Log_File: mysql-relay-log.000001
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes           # Slave这两项必须为Yes
            Slave_SQL_Running: Yes
```

**注意：如果前面对 【主库】做了锁表操作，此时需要： 【 对 Master 解除 table（表）的锁定： "unlock tables;" 】**

### 安装 Nacos

> 所有节点操作步骤一致

[官网教程]:https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html

1）在 GitHub 上下载编译好的 nacos 安装包，下载地址: [Release](https://github.com/alibaba/nacos/releases)

```bash
$ wget https://github.com/alibaba/nacos/releases/download/2.0.4/nacos-server-2.0.4.tar.gz
$ tar -xvf nacos-server-2.0.4.tar.gz -C /usr/local/
```

2）登录 MySQL Master 节点，创建数据库并初始化 [SQL 数据源](https://github.com/alibaba/nacos/blob/master/distribution/conf/nacos-mysql.sql)

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

3）所有节点配置 nacos 后端数据库，在 nacos 的 conf 目录下修改 [application.properties 配置文件](https://github.com/alibaba/nacos/blob/master/distribution/conf/application.properties)

```bash
$ vim /usr/local/nacos/conf/application.properties
#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
# spring.datasource.platform=mysql

### Count of DB:
# db.num=1

### Connect URL of DB:
# db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
# db.user.0=nacos
# db.password.0=nacos

# 根据教程提示进行修改
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://188.188.4.210:3306/nacos_db?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=Asia/Shanghai
db.user.0=nacos
db.password.0=nacos
```

4）在 nacos 的解压目录 `nacos/conf` 目录下，复制 `example` 作为 nacos 的配置文件

```bash
$ cd /usr/local/nacos/conf/
$ cp cluster.conf.example cluster.conf
$ vim cluster.conf
# 修改文件内节点信息，ip:port
188.188.4.210:8848
188.188.4.211:8848
188.188.4.212:8848
```

5）启动服务并测试

```bash
$ sh /usr/local/nacos/bin/startup.sh
/usr/local/java/bin/java -Djava.ext.dirs=/usr/local/java/jre/lib/ext:/usr/local/java/lib/ext  -server -Xms2g -Xmx2g -Xmn1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/nacos/logs/java_heapdump.hprof -XX:-UseLargePages -Dnacos.member.list= -Xloggc:/usr/local/nacos/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/usr/local/nacos/plugins/health,/usr/local/nacos/plugins/cmdb,/usr/local/nacos/plugins/selector -Dnacos.home=/usr/local/nacos -jar /usr/local/nacos/target/nacos-server.jar  --spring.config.additional-location=file:/usr/local/nacos/conf/ --logging.config=/usr/local/nacos/conf/nacos-logback.xml --server.max-http-header-size=524288
nacos is starting with cluster
nacos is starting，you can check the /usr/local/nacos/logs/start.out

$ tail -f /usr/local/nacos/logs/start.out
2022-04-16 17:40:05,214 INFO Nacos is starting...
2022-04-16 17:40:06,215 INFO Nacos is starting...
2022-04-16 17:40:07,217 INFO Nacos is starting...
2022-04-16 17:40:08,235 INFO Nacos is starting...
2022-04-16 17:40:08,282 INFO Nacos started successfully in cluster mode. use external storage

$ tail -f /usr/local/nacos/logs/nacos.log
2022-04-16 17:40:15,140 INFO [Cluster-188.188.4.211:8848] RpcClient init, ServerListFactory = com.alibaba.nacos.core.cluster.remote.ClusterRpcClientProxy$1
2022-04-16 17:40:15,141 INFO [Cluster-188.188.4.211:8848] Try to connect to server on start up, server: {serverIp = '188.188.4.211', server main port = 8848}
2022-04-16 17:40:15,253 INFO [Cluster-188.188.4.211:8848] Success to connect to server [188.188.4.211:8848] on start up, connectionId = 1650102015144_188.188.4.210_16346
2022-04-16 17:40:15,254 INFO [Cluster-188.188.4.211:8848] Register server push request handler:com.alibaba.nacos.common.remote.client.RpcClient$ConnectResetRequestHandler
2022-04-16 17:40:15,254 INFO [Cluster-188.188.4.211:8848] Register server push request handler:com.alibaba.nacos.common.remote.client.RpcClient$$Lambda$790/669957443
```

6）打开浏览器访问 nacos 页面 `http://188.188.4.210:8848/nacos`，默认账密 `nacos/nacos`

![image-20220416174338774](https://cdn.jsdelivr.net/gh/yuikuen/picgo-cdn_images/img/image-20220416174338774.png)

7）配置 systemd 管理 nacos

**注：**虽然系统已配置了 java 环境，并且直接调用 `startup.sh` 也能成功启动，但使用服务 `service` 就无法启动，原因是服务脚本的环境与系统环境变量不能共享

- 方法一：服务文件加载指定的 Java 环境
- 方法二：配置文件指定 Java 环境

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

### 安装 Nginx

```bash
# 下载依赖并编辑安装
$ yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
$ wget http://nginx.org/download/nginx-1.21.4.tar.gz
$ tar -xf nginx-1.21.4.tar.gz && cd nginx-1.21.4
$ ./configure --prefix=/usr/local/nginx 
$ make && make install 

# 配置关机自启
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

$ systemctl enable --now nginx && systemctl status nginx && systemctl status nginx
```

### 负载均衡

- 单节点 Nginx 轮询负载均衡，4.210 安装并配置 Nginx 代理后端 Nacos 服务

```bash
$ cp /usr/local/nginx/conf/nginx.conf /usr/local/nginx/conf/nginx.conf.bak.$(date +%F_%T)
$ vim /usr/local/nginx/conf/nginx.conf
# 增加一行
include /usr/local/nginx/vhosts/*.conf;

$ vim /usr/local/nginx/vhosts/nacos.conf
upstream  nacos-servers {
    server 188.188.4.210:8848;
    server 188.188.4.211:8848;
    server 188.188.4.212:8848;
}

server {
    listen       80;
    server_name  127.0.0.1;
   #server_name  nacos.yuikuen.top; 域名配置，自行选择，可忽略
        
    location /nacos {
        proxy_pass http://nacos-servers;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Real-PORT $remote_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     }
}
```

![image-20220418143641711](https://cdn.jsdelivr.net/gh/yuikuen/picgo-cdn_images/img/image-20220418143641711.png)

