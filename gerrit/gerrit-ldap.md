# Gerrit集成ldap登录

> 之前已介绍过了 Gerrit 的基本安装，并且是采用 http 认证，本次采用的是 OpenLDAP 作用户认证

## 前置环境

配置过程与之前源码安装的一样，不再详细说明，可参考[Gerrit源码安装3.5.x](gerrit-source.md)

## 安装 Gerrit

1)创建目录并下载上传安装包

```bash
$ sudo mkdir -p /usr/local/gerrit
$ cd !$ && sudo wget https://gerrit-releases.storage.googleapis.com/gerrit-3.5.2.war
```

2)执行命令安装并按自身环境选择配置

```bash
$ sudo java -jar gerrit-3.5.2.war init -d review_site

java -jar gerrit-3.5.2.war init -d review_site
Using secure store: com.google.gerrit.server.securestore.DefaultSecureStore
[2022-07-07 11:47:33,095] [main] INFO  com.google.gerrit.server.config.GerritServerConfigProvider : No /usr/local/gerrit/review_site/etc/gerrit.config; assuming defaults

*** Gerrit Code Review 3.5.2
*** 

Create '/usr/local/gerrit/review_site' [Y/n]? 

*** Git Repositories
*** 

Location of Git repositories   [git]: 

*** JGit Configuration
*** 

Auto-configured "receive.autogc = false" to disable auto-gc after git-receive-pack.

*** Index
*** 

Type                           [lucene]: 

*** User Authentication
*** 

Authentication method          [openid/?]: ldap
Git/HTTP authentication        [http/?]: 
LDAP server                    [ldap://localhost]: ldap://188.188.4.140
LDAP username                  : cn=Manager,dc=yuikuen,dc=top
cn=Manager,dc=yuikuen,dc=top's password : 
              confirm password : 
Account BaseDN                 [DC=188,DC=4,DC=140]: ou=People,dc=yuikuen,dc=top
Group BaseDN                   [ou=People,dc=yuikuen,dc=top]: cn=gerritUsers,ou=Group,dc=yuikuen,dc=top
Enable signed push support     [y/N]? n
Use case insensitive usernames [Y/n]? n

*** Review Labels
*** 

Install Verified label         [y/N]? 

*** Email Delivery
*** 

SMTP server hostname           [localhost]: 
SMTP server port               [(default)]: 
SMTP encryption                [none/?]: 
SMTP username                  : 

*** Container Process
*** 

Run as                         [root]: gerrit
Java runtime                   [/usr/local/jdk-11.0.15.1]: 
Copy gerrit-3.5.2.war to review_site/bin/gerrit.war [Y/n]? 
Copying gerrit-3.5.2.war to review_site/bin/gerrit.war

*** SSH Daemon
*** 

Listen on address              [*]: 
Listen on port                 [29418]: 
Generating SSH host key ... rsa... ed25519... ecdsa 256... ecdsa 384... ecdsa 521... done

*** HTTP Daemon
*** 

Behind reverse proxy           [y/N]? 
Use SSL (https://)             [y/N]? 
Listen on address              [*]: 
Listen on port                 [8080]: 
Canonical URL                  [http://m2-debug:8080/]: http://188.188.4.44:8080/

*** Cache
*** 


*** Plugins
*** 

Installing plugins.
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
Reindexed 2 documents in groups index in 0.2s (8.9/s)
Index groups in version 8 is ready
Reindexing changes: Slicing projects: 100% (2/2), done    
Reindexed 0 documents in changes index in 0.5s (0.0/s)
Index changes in version 71 is ready
Reindexing projects:    100% (2/2)
Reindexed 2 documents in projects index in 0.1s (38.5/s)
Index projects in version 4 is ready
Executing /usr/local/gerrit/review_site/bin/gerrit.sh start
Starting Gerrit Code Review: FAILED
error: cannot start Gerrit: exit status 1
Waiting for server on 188.188.4.44:80 ...

$ chown -R gerrit. /usr/local/gerrit
```
注：此处有个小知识点，第一位登录的默认会成为管理员，之后登录的都会是普通用户，请注意！

![20220714105029.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714105029.png)

![20220714105129.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714105129.png)

## 强制分组

> Gerrit 接入 ldap 后，跟 Gitlab 一样，默认是全部用户都可访问登录，在此需求控制到组登录，但按照[官网教程](https://gerrit-review.googlesource.com/Documentation/config-gerrit.html#ldap)或其他教程都未成功，在此贴上配置文件，如有成功的朋友请 @mail 我一下呗。

```bash
[gerrit]
        basePath = git
        canonicalWebUrl = http://188.188.4.44:8080/
        serverId = 4a7439a9-b90e-475d-ae00-d666ce202d50
[container]
        javaOptions = "-Dflogger.backend_factory=com.google.common.flogger.backend.log4j.Log4jBackendFactory#getInstance"
        javaOptions = "-Dflogger.logging_context=com.google.gerrit.server.logging.LoggingContext#getInstance"
        user = root
        javaHome = /usr/local/jdk-11.0.15.1
[index]
        type = lucene
[auth]
        type = LDAP
        gitBasicAuthPolicy = HTTP
[ldap]
        # ldap服务器地址，主从用空格分隔
        server = ldap://188.188.4.140
        # ldap服务器用户名
        username = cn=Manager,dc=yuikuen,dc=top
        # ldap服务器密码
        password = Admin@123
        # ldap包含所有用户账户的根
        accountBase = ou=People,dc=yuikuen,dc=top
        # 用户查询模式，搜索的参数值
        accountPattern = (&(objectClass=inetOrgPerson)(uid=${username}))
        # 用户账户对象的属性名称
        accounFullName = displayName
        # 用户账户对象的属性名称
        accounEmailAddress = mail
        # ldap包含所有组对象的根
        goroupBase = ou=Group,dc=yuikuen,dc=top
        # 组对象的属性名称
        groupName = ${cn} (${gidNumber})
        # 登录时是否获取memberOf账户属性
        fetchMemberOfEagerly = true
        # 所有用户都必须为该组成员才能允许创建或身份验证
        mandatoryGroup = ldap/"gerritUsers"
        # 验证时不区分大小写
        localUsernameToLowerCase = true
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
```
注：修改配置后需要重启服务方可生效

**参考链接：**

- [Section ldap](https://gerrit-review.googlesource.com/Documentation/config-gerrit.html#ldap)
- [gerrit 接入ldap控制到组登录](http://events.jianshu.io/p/c58b425c3288)
- [Gerrit LDAP mandatoryGroup](https://stackoverflow.com/questions/53732683/gerrit-ldap-mandatorygroup)