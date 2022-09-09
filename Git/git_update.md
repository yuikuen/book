# Git Update

> CentOS7.x 更新 git 版本

Centos 7.x 自带的 git 版本为 1.8.3.1，因版本过低，个别应用不兼容，在此进行升级更新；

1）确认当前版本

```bash
$ git --version
git version 1.8.3.1
```

2）配置存储库

启用 Wandisco GIT 存储库，在此之前先写入新 yum 存储库配置文件，在终端输入：

```bash
$ vim /etc/yum.repos.d/wandisco-git.repo
[wandisco-git]
name=Wandisco GIT Repository
baseurl=http://opensource.wandisco.com/centos/7/git/$basearch/
enabled=1
gpgcheck=1
gpgkey=http://opensource.wandisco.com/RPM-GPG-KEY-WANdisco
```

3）导入存储库 GPG 密钥并安装

```bash
$ sudo rpm --import http://opensource.wandisco.com/RPM-GPG-KEY-WANdisco
$ yum install git
```

4）验证 Git 版本

```bash
$ git --version
git version 2.31.1
```