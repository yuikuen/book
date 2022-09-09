# Gerrit GitWeb

> GitWeb 是一个支持在 Web 页面上查看代码以及提交信息的工具，将 GitWeb 工具集成到 Gerrit 中，就可以直接在 Gerrit 的项目列表中查看项目的代码信息

![20220714121640.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714121640.png)
*所上图所示，Gerrit 安装后，默认展示方式为 browse，下面将演示 Gerrit 集成 GitWeb*

## 安装 GitWeb

1）确保 Gerrit 服务正常

```bash
$ sudo netstat -lntp | grep -i gerrit
tcp        0      0 0.0.0.0:29418           0.0.0.0:*               LISTEN      40544/GerritCodeRev 
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      40544/GerritCodeRev
```

2）通过 yum 安装 GitWeb，并设置 Project 目录

```bash
$ sudo yum -y install gitweb httpd

# 配置文件路径
$ sudo ls /etc/ | grep gitweb
gitweb.conf

# 添加projectroot目录，为gerrit安装时定义的git目录路径
$ sudo cat /etc/gitweb.conf |grep -v "#" |grep -Ev "^$"
our $projectroot = "/usr/local/gerrit/review_site/git"
```

## 配置 Httpd

1）GitWeb 是基于 httpd 服务工作的，配置文件 `/etc/httpd/conf.d/git.conf`

```bash
$ sudo cat /etc/httpd/conf.d/git.conf
# 安装后默认的配置文件内容
Alias /git /var/www/git

<Directory /var/www/git>
  Options +ExecCGI
  AddHandler cgi-script .cgi
  DirectoryIndex gitweb.cgi
</Directory>
```

2）修改配置文件，添加 GitWeb 模块文件

```bash
$ sudo cat /etc/httpd/conf.d/git.conf
Alias /gitweb /var/www/git
SetEnv GITWEB_CONFIG /etc/gitweb.conf
<Directory /var/www/git>
  Options +ExecCGI +FollowSymLinks +SymLinksIfOwnerMatch
  AllowOverride All
  order allow,deny
  Allow from all
  AddHandler cgi-script .cgi
  DirectoryIndex gitweb.cgi
</Directory>
```

3）重启服务，并打开浏览器访问测试 http://188.188.4.44/gitweb

```bash
$ sudo systemctl enable httpd;systemctl status httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-07-14 13:54:38 CST; 8min ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 2766 (httpd)
   Status: "Total requests: 5; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─2766 /usr/sbin/httpd -DFOREGROUND
           ├─2767 /usr/sbin/httpd -DFOREGROUND
           ├─2768 /usr/sbin/httpd -DFOREGROUND
           ├─2769 /usr/sbin/httpd -DFOREGROUND
           ├─2770 /usr/sbin/httpd -DFOREGROUND
           ├─2771 /usr/sbin/httpd -DFOREGROUND
           ├─2774 /usr/sbin/httpd -DFOREGROUND
           ├─2775 /usr/sbin/httpd -DFOREGROUND
           └─2776 /usr/sbin/httpd -DFOREGROUND

# httpd默认端口是80，如冲突或其它需要请自行更改
$ sudo cat /etc/httpd/conf/httpd.conf |grep -v "#" |grep -Ev "^$" |grep -i listen
Listen 80
```

![20220714135552.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714135552.png)

## 集成 Gerrit

1）修改 Gerrit 配置文件，增加如下部分

```bash
$ sudo vim ./gerrit/review_site/etc/gerrit.config
[gerrit]
        basePath = git
        canonicalWebUrl = http://188.188.4.44:8080/
        serverId = 22f1c7b9-d041-4e0c-b1c6-c48941b56fef
[container]
        javaOptions = "-Dflogger.backend_factory=com.google.common.flogger.backend.log4j.Log4jBackendFactory#getInstance"
        javaOptions = "-Dflogger.logging_context=com.google.gerrit.server.logging.LoggingContext#getInstance"
        user = gerrit
        javaHome = /usr/local/jdk-11.0.15.1
[index]
        type = lucene
[auth]
        type = LDAP
        gitBasicAuthPolicy = HTTP
        userNameCaseInsensitive = false
[ldap]
        server = ldap://188.188.4.140
        username = cn=Manager,dc=yuikuen,dc=top
        accountBase = ou=People,dc=yuikuen,dc=top
        groupBase = cn=gerritUsers,ou=Group,dc=yuikuen,dc=top
[receive]
        enableSignedPush = false
[sendemail]
        smtpServer = localhost
[sshd]
        listenAddress = *:29418
[httpd]
        listenUrl = http://*:8080/
[cache]
        directory = cache
# 增加gitweb相关配置信息
[gitweb]
        type = gitweb
        cgi = /var/www/git/gitweb.cgi
```

2）修改 git 配置，并重启 Gerrit 服务

```bash
$ git config --file /usr/local/gerrit/review_site/etc/gerrit.config gitweb.cgi /var/www/git/ gitweb.cgi
$ git config --file /usr/local/gerrit/review_site/etc/gerrit.config --unset gitweb.url

$ sudo ./gerrit/review_site/bin/gerrit.sh restart
Stopping Gerrit Code Review: OK
Starting Gerrit Code Review: OK
```

3）重新登录，确认 GitWeb 链接
BROWSE--->Repositories--->Repository Browser--->gitweb 链接

![20220714141626.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714141626.png)

![20220714141824.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714141824.png)

4）配置 GitWeb 访问权限

> 默认只有 Gerrit 管理员用户才有 GitWeb 的访问权限

BROWSE--->Repositories--->All-Projects--->Access--->Reference: refs/meta/config

![20220714143239.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714143239.png)

![20220714143539.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714143539.png)