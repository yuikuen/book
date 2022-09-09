# Gerrit Install(SRC)

> Gerrit 是一个基于 Web 的代码评审工具，建立在 Git 版本控制系统之上。用于代码入库之前对每个提交进行审阅，以确认方案设计和代码实现的质量保证机制。

**官方参考文档如下：**

- [HomePage](https://www.gerritcodereview.com/)
- [Downloads](https://gerrit-releases.storage.googleapis.com/index.html)
- [Docs](https://gerrit-review.googlesource.com/Documentation/)
- [Quickstart](https://gerrit-review.googlesource.com/Documentation/linux-quickstart.html)
- [IssuesList](https://bugs.chromium.org/p/gerrit/issues/list)

**前置条件**

To run the Gerrit service, the following requirement must be met on the host:

- JRE, versions 1.8 or 11 [Download](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
  
  Gerrit is not yet compatible with Java 13 or newer at this time.

## 安装 Java

> 采用源码安装方式，注意上述官方信息 Gerrit 目前尚未与 Java 13 或更新兼容

```bash
$ tar -zxvf jdk-11.0.15.1_linux-x64_bin.tar.gz -C /usr/local/

# 修改环境变量
$ vim /etc/profile.d/jdk11.sh
export JAVA_HOME=/usr/local/jdk-11.0.15.1
export JRE_HOME=$JAVA_HOME
export CLASSPATH=.:$JAVA_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin

$ source /etc/profile && bash
$ java --version
java 11.0.15.1 2022-04-22 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.15.1+2-LTS-10)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.15.1+2-LTS-10, mixed mode)
```

## 安装 Git

> 默认安装的 git 版本为 1.8.3.1，因版本过低可能会导致个别应该不兼容，提前升级

```bash
# 配置Wandisco GIT存储库源文件
$ vim /etc/yum.repos.d/wandisco-git.repo
[wandisco-git]
name=Wandisco GIT Repository
baseurl=http://opensource.wandisco.com/centos/7/git/$basearch/
enabled=1
gpgcheck=1
gpgkey=http://opensource.wandisco.com/RPM-GPG-KEY-WANdisco

# 导入存储库并开始安装，验证版本
$ rpm --import http://opensource.wandisco.com/RPM-GPG-KEY-WANdisco
$ yum -y install git
$ git --version
```

## 安装 MySQL

> 此步骤可选择性安装配置，很多网上的教程都有数据库配置的内容，那是因为 Gerrit 2.x 版本安装时有提示 `SQL Database` 的配置项，但在 Gerrit 3.x 安装的过程已取消了，默认安装的是自带的 H2 数据库，所以在部署完 Gerrit 之后，会发现数据库是空的，其实并不是没有用到，详细可查阅 [官方介绍](https://www.gerritcodereview.com/about.html)

![image-20220705154641686](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220705154641686.png)

网上教程：H2数据库迁移至MySQL数据库，使用工具 [h2tomysql_for_gerrit](https://github.com/yuikuen/h2tomysql_for_gerrit/tree/master/src/com/ucweb/gerrit/tools/converter) 进行转换，再重新索引程序文件，因项目文件都没继续维护，不确定能成功，有兴趣的可自行尝试

Update: 22/07/13 浏览新版本的 [官网介绍](https://gerrit-documentation.storage.googleapis.com/Documentation/3.6.0/config-gerrit.html#accountPatchReviewDb.url) 发现有 `MigrateAccountPatchReviewDb` 数据库迁移参数，在此记录，后续更新测试

![20220713115250.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220713115250.png)

1）安装过程在此略过，使用版本：源码 mysql-boost-5.7.37，仅 Gerrit 2.x 可用

```bash
$ vim /etc/my.cnf
[mysqld]
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/usr/local/mysql/mysql.sock

# 在此增加下述内容并重启服务
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
default-storage-engine=INNODB
skip-character-set-client-handshake

[client]
default_character-set=utf8

$ systemctl restart mysql
```

2）创建数据库&数据库用户并赋权

```sql
mysql>CREATE DATABASE reviewdb CHARACTER SET utf8;
mysql>set global validate_password_policy=0;
mysql>CREATE USER 'gerrit'@'%'IDENTIFIED BY 'Admin@123';
mysql>GRANT ALL PRIVILEGES ON reviewdb.* TO 'gerrit'@'%';
mysql>FLUSH PRIVILEGES;
```

3）在 Gerrit 2.x 安装时，配置 `SQL Database`

```bash
...
*** SQL Database      
***

Database server type           [h2]:mysql
Server hostname                [localhost]:188.188.4.140
Server port                    [default]:3306
Database name                  [db]:reviewdb
Database username              [root]:gerrit
gerrit`s password              :
              confirm password :
```

安装好之后可在配置文件 `gerrit.config` 查看配置信息

## 创建账户

```bash
$ adduser gerrit
$ passwd gerrit
Changing password for user gerrit.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.

$ su - gerrit
$ git config --global core.quotepath false
$ git config --global i18n.logoutputencoding utf8
$ git config --global i18n.commitencoding utf8
$ git config --list
core.quotepath=false
i18n.logoutputencoding=utf8
i18n.commitencoding=utf8

$ exit
$ visudo
$ sudo cat /etc/sudoers |grep gerrit
[sudo] password for gerrit: 
gerrit  ALL=(ALL)       NOPASSWD: ALL
```

## 安装 Gerrit

此处需要注意：本次演示并没使用最新版本，因最前版本移除了 CentOS，具体请自行查阅 [版本介绍](https://www.gerritcodereview.com/releases-readme.html)

以 `java -jar gerrit-x.war init -d $GERRIT_SITE` 方式执行，按提示进行选项配置，后续可通过修改配置文件 `$GERRIT_SITE/etc/gerrit.config` 来更新配置

1）下载安装包并上传，[版本文件](https://gerrit-releases.storage.googleapis.com/index.html)请自行选择，2.x 与 3.x 安装过程并无差异

```bash
$ su - gerrit
$ sudo mkdir -p /usr/local/gerrit
$ cd !$ && sudo wget https://gerrit-releases.storage.googleapis.com/gerrit-3.5.2.war
```

2）执行命令安装并按自身环境选择配置

```bash
$ sudo chown -R gerrit. /usr/local/gerrit
$ java -jar gerrit-3.5.2.war init -d review_site
Using secure store: com.google.gerrit.server.securestore.DefaultSecureStore
[2022-07-05 15:21:51,728] [main] INFO  com.google.gerrit.server.config.GerritServerConfigProvider : No /usr/local/gerrit/review_site/etc/gerrit.config; assuming defaults

*** Gerrit Code Review 3.5.2
*** 
# 是否安装在此目录，选n自动退出安装，默认回车即可
Create '/usr/local/gerrit/review_site' [Y/n]? y

*** Git Repositories
*** 
# Git存储库，默认目录./gerrit/review_site/git
Location of Git repositories   [git]: 

*** JGit Configuration
*** 

Auto-configured "receive.autogc = false" to disable auto-gc after git-receive-pack.

*** Index
*** 

Type                           [lucene]:                 # 设置索引类型，默认即可

*** User Authentication
*** 

Authentication method          [openid/?]: http          # 认证方式，设置http认证
Get username from custom HTTP header [y/N]? n            # 是否自定义http头获取用户名
SSO logout URL                 : 
Enable signed push support     [y/N]? y                  # 启用签名推送支持
Use case insensitive usernames [Y/n]? 

*** Review Labels
*** 

Install Verified label         [y/N]?                    # 安装已验证标签

*** Email Delivery
*** 

SMTP server hostname           [localhost]: smtp.qq.com  # smtp邮箱服务器    
SMTP server port               [(default)]: 465          # 465/994-SSL协议端口，25-非SSL协议端口
SMTP encryption                [none/?]: SSL             # 根据上协议端口定义
SMTP username                  : sinath@vip.qq.com       # 自动发送邮件地址
sinath@vip.qq.com's password   :                         # QQ邮箱此处为授权码，非邮箱密码，如163、企业邮箱即为密码
              confirm password :
*** Container Process
*** 

Run as                         [root]:                   # 运行进程用户，默认为当前用户
Java runtime                   [/usr/lib/jvm/java-11-openjdk-11.0.15.0.9-2.el7_9.x86_64]: 
Copy gerrit-3.5.2.war to review_site/bin/gerrit.war [Y/n]? y
Copying gerrit-3.5.2.war to review_site/bin/gerrit.war

*** SSH Daemon
*** 

Listen on address              [*]:                      # 指定SSH后台服务监听地址
Listen on port                 [29418]:                  # 指定SSH后台服务端口
Generating SSH host key ... rsa... ed25519... ecdsa 256... ecdsa 384... ecdsa 521... done

*** HTTP Daemon
*** 

Behind reverse proxy           [y/N]? y                  # 开启反向代理
Proxy uses SSL (https://)      [y/N]?                    # 询问是否使用https
Subdirectory on proxy server   [/]:                      # 设置子目录位置，默认为根目录即可
Listen on address              [*]: 127.0.0.1            # 设置Gerrit监听的地址，默认为全部
Listen on port                 [8081]:                   # 设置Gerrit监听的端口
Canonical URL                  [http://127.0.0.1/]: http://188.188.4.44  # 设置标准链接地址

*** Cache
*** 


*** Plugins
*** 

Installing plugins.                                      # 插件安装，默认y 
Install plugin codemirror-editor version v3.5.2 [y/N]? y
Installed codemirror-editor v3.5.2
Install plugin commit-message-length-validator version v3.5.2 [y/N]? y
Installed commit-message-length-validator v3.5.2
Install plugin delete-project version v3.5.2 [y/N]? y
Installed delete-project v3.5.2
Install plugin download-commands version v3.5.2 [y/N]? y
Installed download-commands v3.5.2
Install plugin gitiles version v3.5.2 [y/N]? y
Installed gitiles v3.5.2
Install plugin hooks version v3.5.2 [y/N]? y
Installed hooks v3.5.2
Install plugin plugin-manager version v3.5.2 [y/N]? y
Installed plugin-manager v3.5.2
Install plugin replication version v3.5.2 [y/N]? y
Installed replication v3.5.2
Install plugin reviewnotes version v3.5.2 [y/N]? y
Installed reviewnotes v3.5.2
Install plugin singleusergroup version v3.5.2 [y/N]? y
Installed singleusergroup v3.5.2
Install plugin webhooks version v3.5.2 [y/N]? y
Installed webhooks v3.5.2
Initializing plugins.

============================================================================
Welcome to the Gerrit community

Find more information on the homepage: https://www.gerritcodereview.com
Discuss Gerrit on the mailing list: https://groups.google.com/g/repo-discuss
============================================================================
Initialized /usr/local/gerrit/review_site
Init complete, reindexing accounts,changes,groups,projects with: reindex --site-path review_site --threads 1 --index accounts --index changes --index groups --index projects --disable-cache-statsReindexed 0 documents in accounts index in 0.0s (0.0/s)
Index accounts in version 11 is ready
Reindexing groups:      100% (2/2)
Reindexed 2 documents in groups index in 0.3s (7.0/s)
Index groups in version 8 is ready
Reindexing changes: Slicing projects: 100% (2/2), done    
Reindexed 0 documents in changes index in 0.0s (0.0/s)
Index changes in version 71 is ready
Reindexing projects:    100% (2/2)
Reindexed 2 documents in projects index in 0.0s (46.5/s)
Index projects in version 4 is ready
Executing /usr/local/gerrit/review_site/bin/gerrit.sh start
Starting Gerrit Code Review: OK
Waiting for server on 188.188.4.44:80 ... OK
Opening http://188.188.4.44/#/admin/projects/ ...FAILED
Open Gerrit with a JavaScript capable browser:
  http://188.188.4.44/#/admin/projects/
```

3）查看配置文件

```bash
$ sudo cat ./gerrit/review_site/etc/gerrit.config
[gerrit]
	basePath = git
	canonicalWebUrl = http://188.188.4.44
	serverId = 63b3ec1e-a0b4-4420-a3cd-f8620c841d84
[container]
	javaOptions = "-Dflogger.backend_factory=com.google.common.flogger.backend.log4j.Log4jBackendFactory#getInstance"
	javaOptions = "-Dflogger.logging_context=com.google.gerrit.server.logging.LoggingContext#getInstance"
	user = root
	javaHome = /usr/lib/jvm/java-11-openjdk-11.0.15.0.9-2.el7_9.x86_64
[index]
	type = lucene
[auth]
	type = HTTP
	userNameCaseInsensitive = true
[receive]
	enableSignedPush = true
[sendemail]
	smtpServer = smtp.qq.com
	smtpServerPort = 465
	smtpEncryption = SSL
	smtpUser = sinath@vip.qq.com
[sshd]
	listenAddress = *:29418
[httpd]
	listenUrl = proxy-http://127.0.0.1:8081/
[cache]
	directory = cache
```

4）启动配置并设置开机自启

```bash
# 基本操作命令
$ ./gerrit/review_site/bin/gerrit.sh start
$ ./gerrit/review_site/bin/gerrit.sh stop
$ ./gerrit/review_site/bin/gerrit.sh status

$ sudo chmod 755 -R /usr/local/gerrit

# 创建systemd服务文件
$ sudo vim /etc/systemd/system/gerrit.service
[Unit]
Description=gerrit Server
After=syslog.target network.target

[Service]
Type=forking
User=gerrit
Group=gerrit
ExecStart=/usr/local/gerrit/review_site/bin/gerrit.sh start
ExecStop=/usr/local/gerrit/review_site/bin/gerrit.sh stop
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target

$ sudo netstat -ltpn |grep -i gerrit
tcp        0      0 0.0.0.0:29418           0.0.0.0:*               LISTEN      62555/GerritCodeRev 
tcp        0      0 127.0.0.1:8081          0.0.0.0:*               LISTEN      62555/GerritCodeRev

$ sudo ps -au |grep -i gerrit
root      60589  0.0  0.0 191900  4380 pts/0    S    10:11   0:00 su - gerrit
gerrit    60590  0.0  0.0 116876  4840 pts/0    S    10:11   0:00 -bash
root      63015  0.0  0.0 241348  7612 pts/0    S+   10:33   0:00 /usr/bin/sudo -E env LD_LIBRARY_PATH=/opt/rh/devtoolset-9/root/usr/lib64:/opt/rh/devtoolset-9/root/usr/lib:/opt/rh/devtoolset-9/root/usr/lib64/dyninst:/opt/rh/devtoolset-9/root/usr/lib/dyninst:/opt/rh/devtoolset-9/root/usr/lib64:/opt/rh/devtoolset-9/root/usr/lib PATH=/opt/rabbitmq/sbin:/opt/erlang/bin:/usr/local/redis/bin:/opt/rh/devtoolset-9/root/usr/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/usr/local/jdk-11.0.15.1/bin:/usr/local/mysql/bin:/home/gerrit/.local/bin:/home/gerrit/bin scl enable devtoolset-9  'ps' '-au'
gerrit    63016  0.0  0.0 112832  2364 pts/0    S+   10:33   0:00 grep --color=auto -i gerrit
```

## 用户认证

> 反向代理方式可以使用 Nginx 或 Apache
>
> *Gerrit创建管理员用户（Gerrit 默认第一个登录的用户名为超级管理员）*

### Nginx

- listen：监听的端口
- auth_basic：用于登录时弹出验证对话框所显示的内容
- auth_basic_user_file：验证用户名和密码是否匹配的文件
- location 部分：表示当用户访问83端口时，nginx直接将此请求代理到8083端口上，也就是“反向代理”

```bash
$ yum -y install nginx
# 直接修改配置文件
$ vim /etc/nginx/nginx.conf
    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        # 添加下列location跳转根目录反向到本地8081
        location / {
          auth_basic              "Gerrit Code Review";
          auth_basic_user_file    /usr/local/gerrit/gerrit.passwd;
          proxy_pass              http://127.0.0.1:8081;
          proxy_set_header        X-Forwarded-For $remote_addr;
          proxy_set_header        Host $host;
        }

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
```

```bash
# 或者另起配置文件
$ vim /etc/nginx/conf.d/gerrit.conf
server {
     listen *:80;
     server_name 188.188.4.44;
     allow   all;
     deny    all;
     
     auth_basic "Welcom to Gerrit Code Review Site!";
     auth_basic_user_file /usr/local/gerrit/gerrit.passwd;

     location / {
        proxy_pass  http://188.188.4.44:8081;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
     }

     location = /favicon.ico {
        log_not_found off;
        access_log off;
     }
   }
   
$ systemctl enable --now nginx
```

```bash
# 需要跟Nginx配置文件对应目录路径
$ htpasswd -c /usr/local/gerrit/gerrit.passwd admin
New password: 
Re-type new password: 
Adding password for user admin

# 上面是管理员，此为普通用户
$ htpasswd -m /usr/local/gerrit/gerrit.passwd demo
New password: 
Re-type new password: 
Adding password for user demo
```

![image-20220706135723677](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220706135723677.png)

### Apache

```bash
# 先停止Nginx服务
$ systemctl disable --now nginx
$ yum -y install httpd
$ vim /etc/httpd/conf.d/gerrit.conf
<VirtualHost *>
    ServerName 188.188.4.44

    ProxyRequests Off
    ProxyVia Off
    ProxyPreserveHost On

    <Proxy *>
          Order deny,allow
          Allow from all
    </Proxy>

    <Location /login/>
      AuthType Basic
      AuthName "Gerrit Code Review"
      AuthBasicProvider file
      AuthUserFile /gerrit.passwd
      Require valid-user
    </Location>

    AllowEncodedSlashes On
    ProxyPass / http://127.0.0.1:8081/
</VirtualHost>

$ systemctl enable --now httpd

# 与Nginx同理，创建密码
$ touch /gerrit.passwd
$ htpasswd /gerrit.passwd "root"
New password: 
Re-type new password: 
Adding password for user root
$ cat /gerrit.passwd 
root:$apr1$XwPU680a$XX/1KGJViZ4QT7NLrl2/n/
```

![image-20220706140528879](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220706140528879.png)

注意：此处的认证方式取其一即可，账号并不互通，root(Apache) 下并无 admin(Nginx) 的用户信息

**参考链接** [Gerrit - 代码评审工具Gerrit简介与安装 ](https://www.cnblogs.com/anliven/p/11980432.html)

