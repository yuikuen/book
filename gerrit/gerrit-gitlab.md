# Gerrit集成GitLab

> Gerrit 本身提供了 `Code Review` 和 `Git` 仓库的两大功能，但实际上都是使用其他的仓库，如 GitLab 和 GitHub.
> 工作原理：Gerrit 位于最终代码库的前面，用于代码的人工审核和对 CI 任务的触发进行验证。
> Gerrit 集成 GitLab/GitHub 后，Gerrit 上的项目仓库发生变化时，会自动同步到 GitLab/GitHub 上对应的项目仓库中。

注意：Gerrit 的同步操作只能是单向同步（Gerrit-->GitLab），而直接在 GitLab 上项目仓库的变动不会自动同步到 Gerrit 上。因此建议在集成后，所有操作都在 Gerrit 上完成。

## 配置互信

1)将 Gerrit 的公钥添加到 GitLab 的管理员账号内，作用是 Gerrit 服务可通过 SSH 拉取 GitLab 上任意项目代码
注：需要将系统用户 gerrit 和 root 用户的公钥都添加上

```bash
# root
$ ssh-keygen -t rsa -C "sinath@vip.qq.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:0rNwBgwOCw+fWWlBbUcfegNgmELF/PvcieW1BprXJkQ sinath@vip.qq.com
The key's randomart image is:
+---[RSA 2048]----+
|o.o=**ooo .      |
| =.BB+o .+ .     |
|  *o.oo.. +      |
|      .o .E.     |
|      o.S.       |
|      .= o+ .    |
|       o.O = .   |
|        * * =    |
|         . +     |
+----[SHA256]-----+

$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCvrtTpmc0h6dJGq25BPe1Ud66Yt6RVmZ7tqBiHSfRt+x4W9eOO55Y3J5YRj82TaQh9gcLvRYnIMDPzuseAn/XSzT+Be2ll4DIF+OULS2XHvmvZpa+o8lTv84YA9ePMb9BHez5h6Qyi7cfjOutNdC9vrv8UPIvnfuHELCA4e9z75B/+cCdVJ62LPWC7M/ygL2fBMKBDkz8NyUGc1UaoUA4fsJvDA4BR6m9JHySRgaJXBREPbShI+Ex1FKN2Lo4Gqmq3P5tebw5w0z2/rTJTziThGcFSgAdOglrybshzvt+svB0FLXw9qz7dTnG15ZRzMgucPMrhotEca6QHXPkd5TWf sinath@vip.qq.com
```

```bash
# gerrit
$ su - gerrit
$ ssh-keygen -t rsa -C "228003666@qq.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/gerrit/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/gerrit/.ssh/id_rsa.
Your public key has been saved in /home/gerrit/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:V/wVsJundf0KilBywuajdqKnFnckmwuQLERvXaaa+So 228003666@qq.com
The key's randomart image is:
+---[RSA 2048]----+
|..     o     ... |
| .. . +    .  . .|
|o. o o      o.  .|
|+.. =..    . .o..|
|.. + == S .  o.oo|
|  o =o.= .    + o|
|   + ++     ..  .|
| E. *..o . . . . |
| .+*.o  . .   .  |
+----[SHA256]-----+

$ cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCwjbZ8FGr/6agqOs1ELnmOaN5FPLkEobKtSVm0mTC9BWfiwUm7F5p7kP2DUirquPG77tX8Ekgh+tXIC+11Wx7DVkGB+7imRUlOrG6CtwAL2r6RqpCNroS99uA22uAS7MPW897zg/gbbIhe+DhnCjtZwwaBVlYtTVre6MQ6x5nYEE1l2G9cKu5R4cfxviIkzufZvfyEcWUaL4qq97cRkUNqleNiGduEE68gZBqZKxt9c0fQkmeftHn6trdu4Z85twRrP96H9qHU+zMSvtaozAk+RrE+FFsRtuc3gICEpimkUwcf5rNY3G2g74qAoYSLL6cxK5lquECT++7Xg+07rBLp 228003666@qq.com
```

![20220715093627.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220715093627.png)

2)配置 GitLab 访问 Gerrit，让 GitLab 也可 SSH 访问 Gerrit

*Gerrit：188.188.4.140*
```bash
$ su - root
$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@188.188.4.141
/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '188.188.4.141 (188.188.4.141)' can't be established.
ECDSA key fingerprint is SHA256:gyMFA5+P+h1Kxnn1740t6o6YP/waANmiLEXOyd4SEvg.
ECDSA key fingerprint is MD5:c9:e7:28:ec:cf:71:42:2f:d7:24:75:af:c6:e7:17:54.
Are you sure you want to continue connecting (yes/no)? yes
/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@188.188.4.141's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@188.188.4.141'"
and check to make sure that only the key(s) you wanted were added.

[Fri Jul 15 root@hd-dev00 ~]$ ssh 188.188.4.141
Last login: Fri Jul 15 14:02:12 2022
[Fri Jul 15 root@hd-dev01 ~]# exit
logout
Connection to 188.188.4.141 closed.
[Fri Jul 15 root@hd-dev00 ~]$ ls -al ~/.ssh/
total 16
drwx------  2 root root   80 Jul 15 14:08 .
dr-xr-x---. 5 root root  232 Jul 15 13:59 ..
-rw-------  1 root root  395 Jul 15 14:03 authorized_keys
-rw-------  1 root root 1679 Jul 15 14:00 id_rsa
-rw-r--r--  1 root root  399 Jul 15 14:00 id_rsa.pub
-rw-r--r--  1 root root  175 Jul 15 14:08 known_hosts
[Fri Jul 15 root@hd-dev00 ~]$ cat ~/.ssh/known_hosts 
188.188.4.141 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBO4w1+RIxKp0bEDRCYK/7hRCsN0tT4dEzwJ30e9kO61JDpa+hpHSTCZunMc+2jOYxg9so84zUZoKSbsjV2TEl6c=
```

*GitLab：188.188.4.141*
```bash
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:cpClPGqj5jl20Ip+es53c6qk3QNGUBpZ/CxKLJ/Xm9A root@hd-dev01
The key's randomart image is:
+---[RSA 2048]----+
|   .=o  .        |
|   ooo +         |
|  ... O          |
| . o + =         |
|  +.B = S        |
|  .*.* E         |
| .ooo.o o        |
|.o=+=..* .       |
|.+B*.ooo=        |
+----[SHA256]-----+

$ cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDNV8V7gboV9dUti3IEgMXf5647WH9C+VGomMcMYGFQacSZ0t7RNGnqHkWZ3RK6/2yvgH+V9Pz4SQmrdpvgqz6A4BZlHMUMc2BQHLSVY0eiuZzHbxfJE7M0vsSpRgk/JC1GTWyd1viXeGcKk806Pwm0xbj8/cRFc8qVudlE1emDrZGTAK8MkwWSoA+w5E97F2+5pRHkYh4gFaCXKhvkXa6uMdJWoxKumGKuyILVNrBJCAaYv7HrSOmCw6uFc6y3aSzweBvmtgXO83jFgQfZp59s7+FsHX3fmH5lwDn57aJG08S4e+qmZVyem4iMg2bYHI0zGFXwKOQNKadVs2kEW9AB root@hd-dev01
```

```bash
[Fri Jul 15 root@hd-dev01 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@188.188.4.140
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host '188.188.4.140 (188.188.4.140)' can't be established.
ECDSA key fingerprint is SHA256:CVQEnmCI6sx0PzycNRankJID4mtBNtJdmBPldmGgJIM.
ECDSA key fingerprint is MD5:2c:de:ce:2a:a8:b6:41:05:ca:55:48:29:70:63:13:f6.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@188.188.4.140's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@188.188.4.140'"
and check to make sure that only the key(s) you wanted were added.

[Fri Jul 15 root@hd-dev01 ~]# ssh 188.188.4.140
Last login: Fri Jul 15 13:59:15 2022
[Fri Jul 15 root@hd-dev00 ~]$ exit
logout
Connection to 188.188.4.140 closed.
[Fri Jul 15 root@hd-dev01 ~]# cat ~/.ssh/known_hosts 
188.188.4.140 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBG07mvR7TuwsRqB6hU6ZLL9uw6wtw4VuXOYvEzgOdu8gz8DdzwDsQkdTq0aNJu8VASQjjzqDkViULRw7xVWPHZY=
```

3)在 Gerrit 服务器上创建同步 GitLab 的配置文件

```bash
$ su - gerrit
$ sudo vim ~/.ssh/config
Host 188.188.4.141
  # 指向私钥的位置
  IdentityFile ~/.ssh/id_rsa
  PreferredAuthentications publickey
  # 安全设置，匹配提示，仅内网测试使用，生产建议勿设
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```

4)更新 Gerrit 配置文件，设置 `replication` 插件实现仓库的自动同步

```bash
$ cat ./review_site/etc/gerrit.config
[gerrit]
	basePath = git
	canonicalWebUrl = http://188.188.4.140:8080/
	serverId = 1bdba5cd-0aca-458e-abd7-81ec167af7f9
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
[gitweb]
        type = gitweb
	cgi = /var/www/git/gitweb.cgi
[plugins]
        allowRemoteAdmin = true

$ sudo ./review_site/bin/gerrit.sh restart
Stopping Gerrit Code Review: OK
Starting Gerrit Code Review: OK
```

## 创建项目

1)在 GitLab 中创建一个项目
- 关闭 Auto DevOps
- 添加 ReadMe 文件

![20220715171831.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220715171831.png)

![20220715172028.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220715172028.png)

2)在 Gerrit 创建一个空项目并同步 GitLab（注意项目名要与刚创建的 GitLab 项目一致）

![20220715172452.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220715172452.png)

3)删除自动创建的目录，并重新从 GitLab 上复制

```bash
$ cd ./gerrit/review_site/git/ && ls -al
total 0
drwxrwxr-x  5 gerrit gerrit  67 Jul 15 17:24 .
drwxrwxr-x 14 gerrit gerrit 150 Jul 15 11:05 ..
drwxrwxr-x  7 gerrit gerrit 119 Jul 15 11:04 All-Projects.git
drwxrwxr-x  7 gerrit gerrit 119 Jul 15 11:12 All-Users.git
drwxrwxr-x  7 gerrit gerrit 100 Jul 15 17:24 demo.git

$ sudo rm -rf demo.git/
$ git clone --bare git@188.188.4.141:gerrit/demo.git
Cloning into bare repository 'demo.git'...
The authenticity of host '188.188.4.141 (188.188.4.141)' can't be established.
ECDSA key fingerprint is SHA256:gyMFA5+P+h1Kxnn1740t6o6YP/waANmiLEXOyd4SEvg.
ECDSA key fingerprint is MD5:c9:e7:28:ec:cf:71:42:2f:d7:24:75:af:c6:e7:17:54.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '188.188.4.141' (ECDSA) to the list of known hosts.
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (3/3), done.

$ ll demo.git/
total 20
drwxrwxr-x 2 gerrit gerrit    6 Jul 15 17:29 branches
-rw-rw-r-- 1 gerrit gerrit  125 Jul 15 17:29 config
-rw-rw-r-- 1 gerrit gerrit   73 Jul 15 17:29 description
-rw-rw-r-- 1 gerrit gerrit   21 Jul 15 17:29 HEAD
drwxrwxr-x 2 gerrit gerrit 4096 Jul 15 17:29 hooks
drwxrwxr-x 2 gerrit gerrit   21 Jul 15 17:29 info
drwxrwxr-x 4 gerrit gerrit   30 Jul 15 17:29 objects
-rw-rw-r-- 1 gerrit gerrit  103 Jul 15 17:29 packed-refs
drwxrwxr-x 4 gerrit gerrit   31 Jul 15 17:29 refs

$ cat demo.git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = true
[remote "origin"]
	url = git@188.188.4.141:gerrit/demo.git
```

## 配置插件

> 同步功能需要通过 Gerrit 的 `replication` 插件来实现，确认插件是否安装并启用

```bash
$ ls /usr/local/gerrit/review_site/plugins/ | grep replication
replication.jar
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220715173853.png)

1)在 `$GERRIT_SITE/etc` 目录下创建 replication.config 文件，用于代码同步
注意：之后每创建一个新的项目，都需要在该配置文件中添加对应的配置

> `replication.config` 是一个 Git 风格的配置文件，它控制着 replication 插件运行，
> 文件内容由一个或多个 remote 部分组成，每个 remote 又包含一个或多个目标 URL 

```bash
$ sudo vim ./gerrit/review_site/etc/replication.config
[remote "demo"]
projects = demo
url = git@188.188.4.141:gerrit/demo.git
push = +refs/heads/*:refs/heads/*
push = +refs/tags/*:refs/tags/*
push = +refs/changes/*:refs/changes/*
threads = 3
```
