# OpenLDAP配置远程SSH登录

## 安装说明

| 系统版本       | 主机名   | IP 地址      | 角色                        | 备注   |
| -------------- | -------- | ------------ | :-------------------------- | ------ |
| CentOS7.9 Mini | ssd-op01 | 188.188.4.41 | OpenLDAP,Httpd,PhpLDAPAdmin | 服务端 |
| CentOS7.9 Mini | ssd-op02 | 188.188.4.42 | OpenLDAP Client             | 客户端 |
| CentOS7.9 Mini | m2-debug | 188.188.4.44 | Debug                       | 测试机 |

> 上一节已经介绍了 OpenLDAP 的搭建和基本操作，下一步就是要实现客户端服务器可以通过 OpenLDAP 账户进行 SSH 远程登录

## 快速配置

1)客户端服务器安装组件

```bash
$ yum -y install openldap-clients nss-pam-ldapd sssd
```

2)添加客户端服务器到 LDAP 服务，执行 `authconfig` 命令进行获取同步

```bash
# 示例文件
authconfig --enablemkhomedir --enableldap --enableldapauth --ldapserver=ldap://188.188.4.41 --ldapbasedn="dc=yuikuen,dc=top" --enableforcelegacy --disablesssd --disablesssdauth --update  
--enablemkhomedir                                     #创建家目录并配置相关用户家目录的环境变量，建议都采用oddjob软件的mkhomedir模块
--enableldap --enableldapauth --ldapserver=ldap://188.188.4.41 --ldapbasedn="dc=yuikuen,dc=top"  #接入ldapserver
--enableforcelegacy --disablesssd --disablesssdauth   #不采用sssd验证的方式，默认会采用。
--update                                              #对相关配置进行更新

$ authconfig --enableldap --enableldapauth --ldapserver="188.188.4.41" --ldapbasedn="dc=yuikuen,dc=top" --update
```

注：指令修改了 `/etc/nsswitch.conf` ,`/etc/nslcd.conf`以及 `/etc/openldap/ldap.conf` 文件

3)启动客户端服务器的相关服务

```bash
$ systemctl restart nslcd
```

注：如重启服务未生效，请自行重启服务器或检查相关服务是否正常

4)到客户端服务器进行验证

```bash
$ getent passwd | grep -i Demo
rd001:x:10001:10001:Demo [Demo user example]:/home/rd001:/bin/bash
```

5)任意测试机 SSH 远程登录客户端服务器 `ssh-op2` 验证

```bash
[root@m2-debug ~]# ssh rd001@188.188.4.42
rd001@188.188.4.42's password: 
Last login: Fri Jul 22 10:36:06 2022 from 188.188.4.44
Could not chdir to home directory /home/rd001: No such file or directory
/usr/bin/id: cannot find name for group ID 10001
-bash-4.2$ id
uid=10001(rd001) gid=10001 groups=10001
-bash-4.2$ cd /home/
-bash-4.2$ ll
total 0
drwx------. 2 admin admin 83 Jul 15 19:36 admin
```

注：此时登录提示会有提醒未自动生成账户的家目录，可根据实际需求进行配置。建议运维账户统一一个家目录，并设置为只读，不然在实际生产环境下，多个账户登录会生成一堆账户的家目录，这不利于管理；

## 配置解义

> 上面的快速配置其实效果与下面的操作步骤差不多，详细的介绍配置的过程

### NSS 服务配置

1)修改 `/etc/nslcd.conf`

```bash
$ cat /etc/nslcd.conf | grep -v '#' | grep -v '^$'
uid nslcd
gid ldap
uri ldap://188.188.4.41:389              # ldap服务器地址
base dc=yuikuen,dc=top                   # ldap目录树的基准
binddn cn=Manager,dc=yuikuen,dc=top      # ldap管理域
bindpw Admin@#123                        # ldap管理者密码
ssl no
tls_cacertdir /etc/openldap/cacerts
```

2)启动服务并设置开机自启

```bash
$ chmod 600 /etc/nslcd.conf
$ systemctl enable --now nslcd
```

3)配置 `/etc/nsswitch.conf`

```bash
$ cat /etc/nsswitch.conf | grep -v '#' | grep -v '^$'
# 文件添加了sss和ldap的认证
passwd:     files sss ldap
shadow:     files sss ldap
group:      files sss ldap
hosts:      files dns myhostname
bootparams: nisplus [NOTFOUND=return] files
ethers:     files
netmasks:   files
networks:   files
protocols:  files
rpc:        files
services:   files sss
netgroup:   files sss ldap
publickey:  nisplus
automount:  files ldap
aliases:    files nisplus
```

4)客户端服务器测试验证

```bash
$ getent passwd | grep -i Demo
rd001:x:10001:10001:Demo [Demo user example]:/home/rd001:/bin/bash
```

### SSSD 服务配置

1)修改 `/etc/sssd/sssd.conf` 文件，在执行 `authconfig` 时默认生成，如不存在则新建

```bash
$ vim /etc/sssd/sssd.conf
[domain/default]
autofs_provider = ldap
ldap_schema = rfc2307bis
krb5_realm = REDPEAK.COM
ldap_search_base = dc=yuikuen,dc=top
krb5_server = 188.188.4.41
id_provider = ldap 
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldap://188.188.4.41:389
ldap_id_use_start_tls = False
cache_credentials = True
ldap_tls_cacertdir = /etc/openldap/cacerts
[sssd]
services = nss, pam, autofs
domains = default

[nss]
homedir_substring = /home

[pam]

[sudo]

[autofs]

[ssh]

[pac]

[ifp]

[secrets]
```

 2)修改文件权限并设置开机自启动

```bash
$ chmod 600 /etc/sssd/sssd.conf
$ systemctl enable --now sssd
```

### SSH 集成

1)修改配置文件 `/etc/ssh/sshd_config`，让 SSH 通过 pam 认证账号

```bash
PasswordAuthentication yes
UsePAM yes
```

注意：默认使用的是密码认证方式，在集成 SSH 登录时需要确保 PasswordAuthentication yes- 配置为 yes

2)修改配置文件 `/etc/pam.d/sshd`，以确认调用 pam 认证文件

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
# 加入下行后，登录成功后自动创建用户的家目录
session    required     pam_mkhomedir.so
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```

3)修改配置文件 `/etc/pam.d/password-auth`，`/etc/pam.d/system-auth`

> 将pam_sss.so都改为pam_ldap.so

```bash
$ cat /etc/pam.d/password-auth
#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authconfig is run.
auth        required      pam_env.so
auth        required      pam_faildelay.so delay=2000000
auth        sufficient    pam_unix.so nullok try_first_pass
auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
#auth        sufficient    pam_sss.so forward_pass
auth        sufficient    pam_ldap.so use_first_pass
auth        required      pam_deny.so

account     required      pam_unix.so broken_shadow
account     sufficient    pam_localuser.so
account     sufficient    pam_succeed_if.so uid < 1000 quiet
#account     [default=bad success=ok user_unknown=ignore] pam_sss.so
account     [default=bad success=ok user_unknown=ignore] pam_ldap.so
account     required      pam_permit.so

password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
#password    sufficient    pam_sss.so use_authtok
password    sufficient    pam_ldap.so use_authtok


password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
-session     optional      pam_systemd.so
session     optional      pam_mkhomedir.so umask=0077
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so
#session     optional      pam_sss.so
session     optional      pam_ldap.so
```

```bash
cat /etc/pam.d/system-auth
#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authconfig is run.
auth        required      pam_env.so
auth        required      pam_faildelay.so delay=2000000
auth        sufficient    pam_unix.so nullok try_first_pass
auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
#auth        sufficient    pam_sss.so forward_pass
auth        sufficient    pam_ldap.so use_first_pass
auth        required      pam_deny.so

account     required      pam_unix.so broken_shadow
account     sufficient    pam_localuser.so
account     sufficient    pam_succeed_if.so uid < 1000 quiet
#account     [default=bad success=ok user_unknown=ignore] pam_sss.so
account     [default=bad success=ok user_unknown=ignore] pam_ldap.so
account     required      pam_permit.so

password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
#password    sufficient    pam_sss.so use_authtok
password    sufficient    pam_ldap.so use_authtok
password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
-session     optional      pam_systemd.so
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so
#session     optional      pam_sss.so
session     optional      pam_ldap.so
```

4)重启 `ssh/sssd/nslcd` 服务

```bash
$ systemctl restart sshd sssd nslcd
```

到此为止就完成了 OpenLDAP 与 SSH 的集成

5)验证 SSH 登录，登录时系统已自动生成用户的家目录

```bash
[root@m2-debug ~]# ssh rd001@188.188.4.42
rd001@188.188.4.42's password: 
Last login: Fri Jul 22 13:44:24 2022 from 188.188.4.44
/usr/bin/id: cannot find name for group ID 10001
[rd001@ssd-op02 ~]$ ls /home/
admin  rd001
```