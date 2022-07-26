# OpenLDAP配置公钥(PublicKey)远程登录客户机

## 前言说明

前面的章节已介绍了 OpenLDAP 的搭建和配置，虽然实现了 LDAP 账户远程登录服务器，但在实际的生产环境并不推荐多用户直接登录，而是采用密钥登录，下面将演示 OpenLDAP 账户通过密钥登录

| 系统版本       | 主机名   | IP 地址      | 角色                        | 备注   |
| -------------- | -------- | ------------ | :-------------------------- | ------ |
| CentOS7.9 Mini | hd-dev01 | 188.188.4.111 | OpenLDAP,Httpd,PhpLDAPAdmin | 服务端 |
| CentOS7.9 Mini | ssd-op01 | 188.188.4.41 | OpenLDAP Client,openssh-ldap | 客户端 |
| CentOS7.9 Mini | ssd-op02 | 188.188.4.42 | OpenLDAP Client              | 客户端 |
| CentOS7.9 Mini | m2-debug | 188.188.4.44 | Debug                        | 测试机 |

**操作流程：**

- 确保不同的客户端与服务端能通过 LDAP 账户 SSH 实现登录
- OpenLDAP 服务端&客户端(指的都是Linux服务器)都安装 `openssh-ldap` 组件，其中服务端需要配置 Schema 参数
- OpenLDAP 服务端生成密钥文件(请看清楚，此处并不是网上教程的"我们要登录的服务器")
- OpenLDAP 指定用户添加 `objectClass: ldapPublicKey` 属性，配置 `sshPublicKey: LDAP服务端公钥`
- 客户端服务器(Linux服务器)设置关联 LDAP 服务和修改 SSH 配置文件
- 任意客户端(Win/Linux/Mac/Tools)使用 LDAP 账户+ LDAP 服务端私钥文件就可以访问任何一台已关联 LDAP 服务的服务器

**配置解义：**

> 这里需要说明一下上述的操作流程，因为网上大多数的教程，都没注明密钥应该在哪个服务器上生成的，有些只是写着"我们要登录的服务器"。
> 顺便也解释一下，为什么是在 LDAP 服务端生成密钥，而不是在每个服务器生成密钥？下面将简单说明一下：

SSHKey 认证，一般用法就是服务端生成密钥，而用户使用 CRT/Xshell 配置私钥进行访问。是的，OpenLDAP 的常规方法也是一样！一开始我采用的方法其实跟网上教程一样，LDAP 服务端和客户端的服务器都安装 `openssh-ldap` 组件，并确认可互相 SSH 远程登录，之后在指定客户端服务器(就是我们要登录的服务器)生成密钥，并将公钥添加至指定用户，最后使用 Xshell 或任意服务器(未安装任何ldap组件)使用此私钥实现了 PublicKey 远程登录。

实验是成功的，但反过来想，那运维人员就跟之前一样，需要每台服务器都配置密钥，再绑定到指定人员，这样每次要记录好哪个 Key 对应哪个服务器，这给运维工作带来极大的麻烦。下面将以实际环境来演示如何最简便管理。

## 服务端配置

> 确保不同的客户端与服务端能通过 LDAP 账户 SSH 实现登录

1)安装 openssh-ldap 组件[服务器4.111]

```bash
# 由于OpenLDAP默认架构中并没有ldapPublicKey，所以用户无法基于sshkey进行认证，需要自行增加ldapPublicKey相关套件
$ yum -y install openssh-ldap
$ rpm -aql |grep openssh-ldap
/usr/share/doc/openssh-ldap-7.4p1
/usr/share/doc/openssh-ldap-7.4p1/HOWTO.ldap-keys
/usr/share/doc/openssh-ldap-7.4p1/ldap.conf
/usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-openldap.ldif
/usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-openldap.schema
/usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-sun.ldif
/usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-sun.schema
```

2)将配置复制至 Schema 目录，并添加组件

```bash
$ cp /usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-openldap.ldif /etc/openldap/schema/
$ cp /usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-openldap.schema /etc/openldap/schema/

$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openssh-lpk-openldap.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=openssh-lpk,cn=schema,cn=config"
```

## 客户端配置

1)安装 openssh-ldap 组件[服务器4.41]

```bash
$ yum install openssh-ldap
```

2)修改 `ldap.conf` 和 `sshd_config` 文件，支持 PubKey 登录

```bash
$ cp /usr/share/doc/openssh-ldap-7.4p1/ldap.conf /etc/ssh/
$ vim /etc/ssh/ldap.conf
# 未使用TLS,此处改为no
ssl no
uri ldap://188.188.4.111

$ vim /etc/ssh/sshd_config
# 取消注释并按下述修改，此处为通过ssh-ldap-wrapper脚本获取密钥并将其提供给SSH服务验证
AuthorizedKeysCommand /usr/libexec/openssh/ssh-ldap-wrapper
AuthorizedKeysCommandUser nobody
PubkeyAuthentication yes
```

## 密钥登录演示

1)首先确认 SSH 登录情况，除 `m2-debug` 外，都已与 LDAP 进行关联

```bash
[root@ssd-op01 ~]# getent passwd | grep -i rd001
rd001:x:10001:10001:Demo [Demo user example]:/home/rd001:/bin/bash

[root@ssd-op02 ~]# getent passwd | grep -i rd001
rd001:x:10001:10001:Demo [Demo user example]:/home/rd001:/bin/bash

[root@m2-debug ~]# getent passwd | grep -i rd001
[root@m2-debug ~]#
[root@m2-debug .ssh]# ssh rd001@188.188.4.41
The authenticity of host '188.188.4.41 (188.188.4.41)' can't be established.
ECDSA key fingerprint is SHA256:OXR8veTKIYSacRz4/kJ1FF7U89Aoryx9by1WcOFpZ/Y.
ECDSA key fingerprint is MD5:99:a3:d8:77:df:b8:b6:b4:fd:05:08:bc:cc:a0:a9:d0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '188.188.4.41' (ECDSA) to the list of known hosts.
rd001@188.188.4.41's password: 
Last login: Tue Jul 26 11:09:02 2022 from 188.188.4.44
Could not chdir to home directory /home/rd001: No such file or directory
/usr/bin/id: cannot find name for group ID 10001
-bash-4.2$ 
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220726110715.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220726110732.png)

注：就算无关联的客户端都是可以通过 LDAP 用户和密码进行登录的，代表之前 SSH 配置没问题；

2)在客户端服务器 `ssd-op01` 生成密钥文件，并添加到 `rd001`

```bash
[root@ssd-op01 .ssh]# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:YrckWy3IOUOCDboumEdiUx7vqRToFW46E4LIKKSiSU0 root@ssd-op01
The key's randomart image is:
+---[RSA 2048]----+
|  .              |
| . +             |
|...Eo .          |
|B.B ++ o .       |
|@B.B .X S .      |
|@=* o..X o       |
|**.. o. .        |
|..+ .            |
|   .             |
+----[SHA256]-----+
```

3)账户添加 `objectClass: ldapPublicKey` 并添加属性 `sshPublicKey`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220722165257.png)

选择 ldapPublicKey 后会提示输入 sshPublicKey 值，添加 LDAP 公钥就可以

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220725150021047.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220725170736978.png)

4)客户端使用私钥文件，并通过 LDAP 用户进行登录操作

- 使用 XShell 工具

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220726111745.png)

- 已关联的客户端服务器[ssd-op02]

注：如提示错误，请注意查看报错信息，一般为权限问题，`chmod 600 id_rsa` 可解决

```bash
[root@ssd-op02 ~]# cd ~/.ssh/
[root@ssd-op02 .ssh]# ll
total 8
-rw------- 1 admin admin 1679 Jul 26 10:27 id_rsa
-rw-r--r-- 1 root  root   349 Jul 25 16:38 known_hosts
[root@ssd-op02 .ssh]# ssh -i id_rsa rd001@188.188.4.41
Last login: Tue Jul 26 11:16:23 2022 from 100.20.0.217
Could not chdir to home directory /home/rd001: No such file or directory
/usr/bin/id: cannot find name for group ID 10001
-bash-4.2$ 

# 注意：反过来未安装openssh-ldap的就算使用私钥也只会变成密码登录
[root@ssd-op01 .ssh]# ssh -i id_rsa rd001@188.188.4.42
rd001@188.188.4.42's password: 
Last login: Tue Jul 26 11:23:44 2022 from 188.188.4.44
Could not chdir to home directory /home/rd001: No such file or directory
/usr/bin/id: cannot find name for group ID 10001
-bash-4.2$
```

- 未关联的客户端服务器[m2-debug]

```bash
[root@m2-debug admin]# pwd
/home/admin
[root@m2-debug admin]# ssh -i /home/admin/id_rsa rd001@188.188.4.41
The authenticity of host '188.188.4.41 (188.188.4.41)' can't be established.
ECDSA key fingerprint is SHA256:OXR8veTKIYSacRz4/kJ1FF7U89Aoryx9by1WcOFpZ/Y.
ECDSA key fingerprint is MD5:99:a3:d8:77:df:b8:b6:b4:fd:05:08:bc:cc:a0:a9:d0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '188.188.4.41' (ECDSA) to the list of known hosts.
Last login: Tue Jul 26 11:19:40 2022 from 188.188.4.42
Could not chdir to home directory /home/rd001: No such file or directory
/usr/bin/id: cannot find name for group ID 10001
-bash-4.2$ exit
logout
Connection to 188.188.4.41 closed.

# 无关联并且没安装openssh-ldap的也是一样变成密码登录
[root@m2-debug admin]# ssh -i /home/admin/id_rsa rd001@188.188.4.42
The authenticity of host '188.188.4.42 (188.188.4.42)' can't be established.
ECDSA key fingerprint is SHA256:eu7BvPm1Pf7tUVwdP6RjzkCUwUnls1G5PODVP+lSenc.
ECDSA key fingerprint is MD5:1e:dc:01:6e:dd:4e:6e:20:e7:15:63:04:97:1a:29:b6.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '188.188.4.42' (ECDSA) to the list of known hosts.
rd001@188.188.4.42's password: 
Could not chdir to home directory /home/rd001: No such file or directory
/usr/bin/id: cannot find name for group ID 10001
-bash-4.2$ 
```

经过三种方法演示，代表 `PublicKey` 集成 LDAP 已成功，由其是最后的演示，可以看到未配对与已配对密钥的访问提示。
虽然达到了我们的目的，但我们需要每台服务器都要创建密钥，再根据用户需求进行发放，这反而增加了工作量。

## 解决方案

> 注：此为我现管理的方法，如觉得有需要，可根据上述进行操作
> 提前将所有客户端服务器安装 `openssh-ldap` 组件

1)LDAP 服务端[hd-dev01]生成密钥

```bash
[root@hd-dev01 ~]# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:3xpWRgLvKXcMeogzNPvBekdY9o8fg1ItiaOaQHyASDg root@hd-dev01
The key's randomart image is:
+---[RSA 2048]----+
|..      .        |
|E. .     o       |
|... . o   * .    |
|   . o = B O o   |
|    o * S O X .  |
|   . . = O B =   |
|    . . + * + +  |
|     . + o + . o |
|      o   .   .  |
+----[SHA256]-----+
[root@hd-dev01 ~]# cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDdWMI6Dw+T75VYv8tUN96pELBeVKM2sisJ8z7/FlAGPqnHvp2c5pgNKaibYbyqW7e7aqUe0+/aghNeooXQNpmnEIfCNEoOFJXuRbEN1nPLt0wztKhdxWVs3A2HIeu8U/pfq/ZSIIfBuPpKgkIsoEdQUuIeHWSj0Oq0zpOx8TCvrj6+idI+B9Ti40GcEdvIjAG/7Whh6MSklVSAjUln+d/pihwwSEkvdKurnFUiBiVe7uQM93i6le0oVHfv7DHiHS+clm7+v0GV1x75TWFgVHf8dMazxW9f3UgTZxzXRU5BdDiNJFuoEXTKMVFw7W2GKsDzmU61LaIRWREpsdr5rLdn root@hd-dev01
```

2)将公钥添加至指定用户

> 为了验证效果，在此另创建新用户，方法同理于 `Web-Ui` 操作

```bash
[root@hd-dev01 myldif]# ll
total 16
-rw-r--r--. 1 root root 402 Jul 23 11:44 base.ldif
-rw-r--r--. 1 root root 876 Jul 23 11:40 changedomain.ldif
-rw-r--r--. 1 root root  73 Jul 23 11:42 ldap_log.ldif
-rw-r--r--. 1 root root 887 Jul 23 11:46 user.ldif
[root@hd-dev01 myldif]# cp user.ldif add_user.ldif
[root@hd-dev01 myldif]# vim add_user.ldif 
[root@hd-dev01 myldif]# cat add_user.ldif 
dn: uid=op001,ou=op,ou=People,dc=yuikuen,dc=top
cn: op001
uid: op001
sn: op001
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
objectClass: ldapPublicKey
sshPublicKey: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDdWMI6Dw+T75VYv8tUN96pELBeVKM2sisJ8z7/FlAGPqnHvp2c5pgNKaibYbyqW7e7aqUe0+/aghNeooXQNpmnEIfCNEoOFJXuRbEN1nPLt0wztKhdxWVs3A2HIeu8U/pfq/ZSIIfBuPpKgkIsoEdQUuIeHWSj0Oq0zpOx8TCvrj6+idI+B9Ti40GcEdvIjAG/7Whh6MSklVSAjUln+d/pihwwSEkvdKurnFUiBiVe7uQM93i6le0oVHfv7DHiHS+clm7+v0GV1x75TWFgVHf8dMazxW9f3UgTZxzXRU5BdDiNJFuoEXTKMVFw7W2GKsDzmU61LaIRWREpsdr5rLdn root@hd-dev01
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
displayName: 袁工
title: Sys_Engineer
mail: op001@yuikuen.top
shadowMin: 0
ShadowMax: 99999
shadowWarning: 7
loginShell: /bin/bash
gecos: Demo [Demo user example]
uidNumber: 20001
gidNumber: 20001
homeDirectory: /home/op001
[root@hd-dev01 myldif]# ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f add_user.ldif 
adding new entry "uid=op001,ou=op,ou=People,dc=yuikuen,dc=top"
```

3)之后各测试机使用该私钥进行登录

```bash
[root@m2-debug .ssh]# chmod 600 id_rsa 
[root@m2-debug .ssh]# ssh -i id_rsa op001@188.188.4.41
The authenticity of host '188.188.4.41 (188.188.4.41)' can't be established.
ECDSA key fingerprint is SHA256:OXR8veTKIYSacRz4/kJ1FF7U89Aoryx9by1WcOFpZ/Y.
ECDSA key fingerprint is MD5:99:a3:d8:77:df:b8:b6:b4:fd:05:08:bc:cc:a0:a9:d0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '188.188.4.41' (ECDSA) to the list of known hosts.
Last login: Tue Jul 26 13:48:28 2022 from 188.188.4.42
Could not chdir to home directory /home/op001: No such file or directory
/usr/bin/id: cannot find name for group ID 10001
-bash-4.2$ id
uid=20001(op001) gid=10001 groups=10001
-bash-4.2$ exit
logout
Connection to 188.188.4.41 closed.
[root@m2-debug .ssh]# ssh -i id_rsa op001@188.188.4.42
The authenticity of host '188.188.4.42 (188.188.4.42)' can't be established.
ECDSA key fingerprint is SHA256:eu7BvPm1Pf7tUVwdP6RjzkCUwUnls1G5PODVP+lSenc.
ECDSA key fingerprint is MD5:1e:dc:01:6e:dd:4e:6e:20:e7:15:63:04:97:1a:29:b6.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '188.188.4.42' (ECDSA) to the list of known hosts.
Last login: Tue Jul 26 13:48:35 2022 from 188.188.4.42
Could not chdir to home directory /home/op001: No such file or directory
/usr/bin/id: cannot find name for group ID 10001
-bash-4.2$ id
uid=20001(op001) gid=10001 groups=10001
-bash-4.2$ exit
logout
Connection to 188.188.4.42 closed.
```

最终实现的效果是：后期只维护 LDAP 服务端的密钥，将其私钥供到指定用户，无需创建并记录多台服务器的密钥，减少维护成本

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220726135501.png)


## 参考链接

- [linux 利用LDAP身份集中认证](https://www.cnblogs.com/dufeixiang/p/11624210.html)
- [ubuntu20.04 + OpenLdap 实现企业运维账户管理系统](https://blog.51cto.com/u_13994871/3788958)
- [ubuntu20.04 + OpenLdap 实现企业运维账户管理系统 （下）](https://blog.51cto.com/u_13994871/5216312)
- [CentOS7 配置OpenLDAP（一） 配置OpenLDAP服务单节点模式并实现服务器登录管理常见场景](https://blog.csdn.net/oyym_mv/article/details/94404663)