# Gerrit安装配置GitWeb

> GitWeb 是一个支持在 Web 页面上查看代码以及提交信息的工具，将 GitWeb 工具集成到 Gerrit 中，就可以直接在 Gerrit 的项目列表中查看项目的代码信息

## 安装 GitWeb

1)确保 Gerrit 服务正常

```bash
$ sudo netstat -lntp | grep -i gerrit
tcp        0      0 0.0.0.0:29418           0.0.0.0:*               LISTEN      40544/GerritCodeRev 
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      40544/GerritCodeRev
```

2)通过 yum 安装 GitWeb，并设置 Project 目录

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

1)GitWeb 是基于 httpd 服务工作的，配置文件 `/etc/httpd/conf.d/git.conf`

```bash

```
