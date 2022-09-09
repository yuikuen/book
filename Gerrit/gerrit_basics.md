# Gerrit Basics

> Gerrit 基本使用方法

## 用户管理

Gerrit 是基于群组来进行权限控制的，不同的群组具有不同的权限，每个用户属于一个或多个群组

Gerrit 系统自带群组：

- Anonymous Users：所有用户自动属于该群组，默认只有 Read 权限
- Change Owner：某个提交的拥有者，具备所属变更的权限
- Project Owners：项目拥有者，具备所属项目的权限
- Registered Users：所有成功登录的用户自动属于该群组，具备投票权限（CodeReview +1-1）

Gerrit预先定义的群组：

- Administartors：该群组的成员可以管理所有项目和系统配置
- Non-Interactive Users：该群组的成员可以通过界面进行操作，一般用于和第三方系统集成
- Service Users：对 Gerrit 执行批处理操作的用户

## 进程管理

```bash
$ ./gerrit/review_site/bin/gerrit.sh 
Usage: gerrit.sh {start|stop|restart|check|status|run|supervise|threads} [-d site]

$ sudo systemctl start|stop|restart gerrit
$ sudo systemctl status gerrit
```

## 日志管理

```bash
$ pwd
/usr/local/gerrit/review_site/logs

$ ls -al
total 276
drwxr-xr-x  2 gerrit gerrit    147 Jul 14 17:12 .
drwxr-xr-x 14 gerrit gerrit    150 Jul 14 10:46 ..
-rwxr-xr-x  1 gerrit gerrit      0 Jul 14 10:46 delete_log
-rwxr-xr-x  1 gerrit gerrit  48407 Jul 14 17:13 error_log
-rwxr-xr-x  1 gerrit gerrit      0 Jul 14 10:46 gc_log
-rw-r--r--  1 gerrit gerrit      5 Jul 14 17:12 gerrit.pid
-rw-r--r--  1 gerrit gerrit     16 Jul 14 17:12 gerrit.run
-rwxr-xr-x  1 gerrit gerrit 223097 Jul 14 17:12 httpd_log
-rwxr-xr-x  1 gerrit gerrit      0 Jul 14 10:46 replication_log
-rwxr-xr-x  1 gerrit gerrit      0 Jul 14 10:46 sshd_log
```

## war 包命令

> War 包内拥有很多可执行命令

```bash
$ java -jar gerrit-3.5.2.war
Gerrit Code Review 
usage: java -jar gerrit-3.5.2.war command [ARG ...]

The most commonly used commands are:
  init            Initialize a Gerrit installation
  reindex         Rebuild the secondary index
  daemon          Run the Gerrit network daemons
  version         Display the build version number
  passwd          Set or change password in secure.config

  ls              List files available for cat
  cat FILE        Display a file from the archive

$ java -jar gerrit-3.5.2.war init -h
init [--batch (-b)] [--delete-caches] [--dev] [--help (-h)] [--install-all-plugins] [--install-plugin VAL] [--list-plugins] [--no-auto-start] [--reindex-threads N] [--secure-store-lib VAL] [--show-stack-trace] [--site-path (-d) VAL] [--skip-all-downloads] [--skip-download VAL] [--skip-plugins]

 --batch (-b)           : Batch mode; skip interactive prompting (default:
                          false)
 --delete-caches        : Delete all persistent caches without asking (default:
                          false)
 --dev                  : Setup site with default options suitable for
                          developers (default: false)
 --help (-h)            : display this help text (default: true)
 --install-all-plugins  : Install all plugins from war without asking (default:
                          false)
 --install-plugin VAL   : Install given plugin without asking
 --list-plugins         : List available plugins (default: false)
 --no-auto-start        : Don't automatically start daemon after init (default:
                          false)
 --reindex-threads N    : Number of threads to use for reindex after init
                          (default: 1)
 --secure-store-lib VAL : Path to jar providing SecureStore implementation class
 --show-stack-trace     : display stack trace on failure (default: false)
 --site-path (-d) VAL   : Local directory containing site data
 --skip-all-downloads   : Don't download libraries (default: false)
 --skip-download VAL    : Don't download given library
 --skip-plugins         : Don't install plugins (default: false)
```