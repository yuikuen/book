# OpenLDAP Client

> OpenLDAP 基本搭建后，下一步就是要实现客户端通过 LDAP 账户进行 SSH 远程登录

## 安装说明

| 主机名 |IP 地址|角色|备注|
|--|--|--|--|
|ssd-dev01|188.188.4.111|OpenLDAP|OpenLDAP 服务端-服务器|
|ssd-dev02|188.188.4.112|LDAP Client|LDAP 客户端-服务器(Linux)|
|win/Mac-pc|100.20.0.250|Client-PC|个人电脑(客户端)-工作站|

## 快速配置

1）客户端服务器安装组件

> 演示环境为 LDAP 客户端(Linux) ssd-dev02 188.188.4.112

```bash
$ yum -y install openldap-clients nss-pam-ldapd
```

2）客户端服务器执行 `authconfig` 命令加载同步 LDAP 服务信息

```bash
$ authconfig --enablemkhomedir --enableldap --enableldapauth --ldapserver=ldap://188.188.4.111 --ldapbasedn="dc=yuikuen,dc=top" --enableforcelegacy --disablesssd --disablesssdauth --update
```

**命令解义**

> 注意：指令修改了  `/etc/nsswitch.conf` ,`/etc/nslcd.conf` 以及 `/etc/openldap/ldap.conf` 文件

```bash
--enablemkhomedir                                                                          # 创建目录并配置用户目录的环境变量
--enableldap --enableldapauth --ldapserver=ldap://ldap-ip --ldapbasedn="dc=example,dc=com" # 接入ldapserver服务
--enableforcelegacy --disablesssd --disablesssdauth                                        # 不采用sssd验证方式，默认会采用
--update                                                                                   # 对相关配置进行更新
```

3）启动客户端的相关服务

```bash
$ systemctl restart nslcd
$ systemctl enable nslcd; systemctl status nslcd
● nslcd.service - Naming services LDAP client daemon.
   Loaded: loaded (/usr/lib/systemd/system/nslcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-07-30 17:38:03 CST; 4s ago
     Docs: man:nslcd(8)
           man:nslcd.conf(5)
 Main PID: 1817 (nslcd)
   CGroup: /system.slice/nslcd.service
           └─1817 /usr/sbin/nslcd

Jul 30 17:38:03 ssd-dev01 systemd[1]: Starting Naming services LDAP client daemon....
Jul 30 17:38:03 ssd-dev01 nslcd[1817]: version 0.8.13 starting
Jul 30 17:38:03 ssd-dev01 nslcd[1817]: accepting connections
Jul 30 17:38:03 ssd-dev01 systemd[1]: Started Naming services LDAP client daemon..
```

4）本地验证，查看是否获取到 LDAP 账户信息

```bash
$ getent passwd | grep -i ldapusers
op001:x:1000:1000:System Manager:/home/ldapusers:/bin/bash
rd001:x:2000:2000:General User:/home/ldapusers:/bin/bash
```

5）个人电脑 SSH 远程登录验证

> LDAP 服务端也是服务器，也能当作客户端服务器

```bash
# 个人电脑使用LDAP账户和Xshell远程登录LDAP服务端
Xshell 7 (Build 0098)
Copyright (c) 2020 NetSarang Computer, Inc. All rights reserved.

Type `help' to learn how to use Xshell prompt.
[C:\~]$ 

Connecting to 188.188.4.111:22...
Connection established.
To escape to local shell, press 'Ctrl+Alt+]'.

WARNING! The remote SSH server rejected X11 forwarding request.
Creating directory '/home/ldapusers'.
/usr/bin/id: cannot find name for group ID 1000
[Sat Jul 30 op001@ssd-dev01 ~]$
```

```bash
# 服务器之间也可通过LDAP账户SSH来远程登录
[Sat Jul 30 root@ssd-dev02 ~]# ssh rd001@188.188.4.111
The authenticity of host '188.188.4.111 (188.188.4.111)' can't be established.
ECDSA key fingerprint is SHA256:EBqNE3wcKs7KQiaTvyVT/mib+vlacvZh3m+M5q2NKeM.
ECDSA key fingerprint is MD5:43:ea:77:e7:35:2e:32:64:bb:96:96:3d:c1:4f:68:1a.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '188.188.4.111' (ECDSA) to the list of known hosts.
rd001@188.188.4.111's password: 
Could not chdir to home directory /home/ldapusers: Permission denied
/usr/bin/id: cannot find name for group ID 2000
-bash: /home/ldapusers/.bash_profile: Permission denied
[Sat Jul 30 rd001@ssd-dev01 /]$
```

注：错误提示为未找到用户的ID信息，并且创建家目录(成功/失败-权限不足)，建议统一目录或不执行，否则会产生过多的账户家目录，默认权限为只读

错误原因：

1. 前面执行了 `--enablemkhomedir` 开启创建家目录的功能导致(可忽略不执行)
2. 创建家目录失败，权限不足导致，因为默认 LDAP 账户无管理员权限

## 配置解义

> 下面将讲解上述具体修改的内容，详细的介绍配置过程

### NSS 服务配置

1）`/etc/nslcd.conf`

```bash
$ cat /etc/nslcd.conf | grep -v '#' | grep -v '^$'
uid nslcd
gid ldap
uri ldap://188.188.4.111               # ldap服务器地址
base dc=yuikuen,dc=top                 # ldap目录树基准
binddn cn=Manager,dc=yuikuen,dc=top    # ldap管理域
bindpw Admin@123                       # ldap管理者密码
ssl no
tls_cacertdir /etc/openldap/cacerts
```

2）`/etc/nsswitch.conf`

```bash
$ cat /etc/nsswitch.conf | grep -v '#' | grep -v '^$'
# 通过authconfig命令更新ldap的认证
passwd:     files ldap
shadow:     files ldap
group:      files ldap
hosts:      files dns myhostname
bootparams: nisplus [NOTFOUND=return] files
ethers:     files
netmasks:   files
networks:   files
protocols:  files
rpc:        files
services:   files
netgroup:   files ldap
publickey:  nisplus
automount:  files ldap
aliases:    files nisplus
```

3）`/etc/openldap/ldap.conf`

```bash
$ cat /etc/openldap/ldap.conf | grep -v '#' | grep -v '^$'
TLS_CACERTDIR /etc/openldap/cacerts
SASL_NOCANON	on
URI ldap://188.188.4.111
BASE dc=yuikuen,dc=top
```

注：此处并没有 SSSD 的服务配置，如需请自行百度进行配置

### SSH 集成

1）修改配置文件 `/etc/ssh/sshd_config`，让 SSH 通过 pam 认证

```bash
PasswordAuthentication yes
UsePAM yes
```

2）修改配置文件 `/etc/pam.d/sshd`，以确认调用 pam 认证文件

```bash
$ cat /etc/pam.d/sshd 
#%PAM-1.0
auth	   required	pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# 增加下述，认证成功后自动创建用户的家目录
session    required     pam_mkhomedir.so
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```

3）修改配置文件 `/etc/pam.d/password-auth`，`/etc/pam.d/system-auth`

> 通过authconfig命令自动加载pam_ldap.so参数内容

```bash
$ cat /etc/pam.d/password-auth
#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authconfig is run.
auth        required      pam_env.so
auth        required      pam_faildelay.so delay=2000000
auth        sufficient    pam_unix.so nullok try_first_pass
auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
auth        sufficient    pam_ldap.so use_first_pass
auth        required      pam_deny.so

account     required      pam_unix.so broken_shadow
account     sufficient    pam_localuser.so
account     sufficient    pam_succeed_if.so uid < 1000 quiet
account     [default=bad success=ok user_unknown=ignore] pam_ldap.so
account     required      pam_permit.so

password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    sufficient    pam_ldap.so use_authtok


password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
-session     optional      pam_systemd.so
session     optional      pam_mkhomedir.so umask=0077
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so
session     optional      pam_ldap.so
```

```bash
$ cat /etc/pam.d/system-auth
#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authconfig is run.
auth        required      pam_env.so
auth        required      pam_faildelay.so delay=2000000
auth        sufficient    pam_unix.so nullok try_first_pass
auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
auth        sufficient    pam_ldap.so use_first_pass
auth        required      pam_deny.so

account     required      pam_unix.so broken_shadow
account     sufficient    pam_localuser.so
account     sufficient    pam_succeed_if.so uid < 1000 quiet
account     [default=bad success=ok user_unknown=ignore] pam_ldap.so
account     required      pam_permit.so

password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    sufficient    pam_ldap.so use_authtok
password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
-session     optional      pam_systemd.so
session     optional      pam_mkhomedir.so umask=0077
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so
session     optional      pam_ldap.so
```