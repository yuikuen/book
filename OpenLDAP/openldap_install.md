# OpenLDAP Install

## 版本环境

> 为了简化 OpenLDAP 的安装复杂程序，直接通过 YUM 源安装，需要定制的才使用编译方式安装，另开篇讲解

- System：CentOS7.9.2009 Minimal
- OpenLDAP：openldap-2.4.44-25.el7_9.x86_64

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装组件

1）服务端安装组件

```bash
$ yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel
```

## 初始化配置

1）使用默认的模板文件创建数据库

```bash
$ cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
$ chown ldap:ldap -R /var/lib/ldap /etc/openldap
$ chmod 700 -R /var/lib/ldap
```
注：`/var/lib/ldap` 为 BerkeleyDB 数据库默认存储的路径

2）启动服务进程，检查是否安装成功

```bash
$ systemctl enable --now slapd && systemctl status slapd
$ netstat -anpl|grep 389
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      1634/slapd          
tcp6       0      0 :::389                  :::*                    LISTEN      1634/slapd 

$ ps aux | grep slapd | grep -v grep
ldap      12684  0.0  0.5 905940 48700 ?        Ssl  Jul27   0:00 /usr/sbin/slapd -u ldap -h ldapi:/// ldap:///
```
LDAP 服务的默认端口为 389，此端口采用明文传输数据，所以建议通过配置 CA 以及结合 TLS/SASL 实现数据加密传传输，默认端口为 636

## 自定义配置

**通过上述操作已完成 LDAP 基本安装，但里面是没有账户、组织等配置参数**

### 导入基本文件

> Schema 类似数据库表，定义了字段名和类型等参数值

1）导入已存在的 ldif 文件

```bash
$ pwd
/etc/openldap/schema
$ ls -al | grep .ldif
-r--r--r--. 1 ldap ldap  2036 Feb 24 01:11 collective.ldif
-r--r--r--. 1 ldap ldap  1845 Feb 24 01:11 corba.ldif
-r--r--r--. 1 ldap ldap 20612 Feb 24 01:11 core.ldif
-r--r--r--. 1 ldap ldap 12006 Feb 24 01:11 cosine.ldif
-r--r--r--. 1 ldap ldap  4842 Feb 24 01:11 duaconf.ldif
-r--r--r--. 1 ldap ldap  3330 Feb 24 01:11 dyngroup.ldif
-r--r--r--. 1 ldap ldap  3481 Feb 24 01:11 inetorgperson.ldif
-r--r--r--. 1 ldap ldap  2979 Feb 24 01:11 java.ldif
-r--r--r--. 1 ldap ldap  2082 Feb 24 01:11 misc.ldif
-r--r--r--. 1 ldap ldap  6809 Feb 24 01:11 nis.ldif
-r--r--r--. 1 ldap ldap  3308 Feb 24 01:11 openldap.ldif
-r--r--r--. 1 ldap ldap  6904 Feb 24 01:11 pmi.ldif
-r--r--r--. 1 ldap ldap  4570 Feb 24 01:11 ppolicy.ldif

$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f cosine.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f nis.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f collective.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f corba.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f core.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f duaconf.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f dyngroup.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f inetorgperson.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f java.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f misc.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f openldap.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f pmi.ldif 
$ ldapadd -Y EXTERNAL -H ldapi:/// -D "cn=config" -f ppolicy.ldif
```
### 改域及配置权限

1）使用 `slappasswd` 生成管理员的密文密码

```bash
$ slappasswd -s Admin@123
{SSHA}+IqqdRVpbUXCbre3dTrtGKt+zZKak9Ef
```

2）修改自定义域，并更新数据库信息

```bash
$ cat changedomain.ldif
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=Manager,dc=yuikuen,dc=top" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=yuikuen,dc=top

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=yuikuen,dc=top

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}+IqqdRVpbUXCbre3dTrtGKt+zZKak9Ef

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by dn="cn=Manager,dc=yuikuen,dc=top" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=Manager,dc=yuikuen,dc=top" write by * read
```

3）执行 `ldapmodify` 修改命令刷新数据库信息

```bash
$ ldapmodify -Y EXTERNAL -H ldapi:/// -f changedomain.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}monitor,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"
```

4）通过 `slaptest` 查看验证是否有问题

```bash
$ slaptest -u
config file testing succeeded
```

### 配置日志功能 

> LDAP 默认未开启日志功能，需要单独配置

1）创建配置参数

```bash
$ cat ldap_log.ldif
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats
```

2）执行 `ldapmodify` 修改命令刷新配置

```bash
$ ldapmodify -Y EXTERNAL -H ldapi:/// -f ldap_log.ldif
```

3）使用系统自带的 `rsyslog` 进行日志输出

```bash
$ vim /etc/rsyslog.conf +74
...
local4.*               /var/log/openldap.log

$ vim /etc/logrotate.d/slapd
/var/log/openldap.log {
    rotate 14
    size 10M
    missingok
    compress
    copytruncate
}

$ systemctl restart rsyslog
```

### 创建基础目录树

> 虽然已配置好 LDAP 的基本服务参数，但里面还是什么都没有的。下面将创建一个基础架构的组织(Company)，并在其下创建名为 `Manager` 的组织角色，和两个组织单元 `People/Group`

1）创建一个 ldif 文件，生成基础的目录树

```bash
$ cat base.ldif
dn: dc=yuikuen,dc=top
o: yuikuen top
dc: yuikuen
objectClass: top
objectClass: dcObject
objectclass: organization

dn: cn=Manager,dc=yuikuen,dc=top
cn: Manager
objectClass: organizationalRole
description: LDAP Manager

dn: ou=People,dc=yuikuen,dc=top
ou: People
objectClass: top
objectClass: organizationalUnit

dn: ou=Group,dc=yuikuen,dc=top
ou: Group
objectClass: top
objectClass: organizationalUnit
```

2）执行 `ldapadd` 创建命令

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f base.ldif
adding new entry "dc=yuikuen,dc=top"

adding new entry "cn=Manager,dc=yuikuen,dc=top"

adding new entry "ou=People,dc=yuikuen,dc=top"

adding new entry "ou=Group,dc=yuikuen,dc=top"
```

3）执行 `ldapsearch` 命令来检查内容

```bash
$ ldapsearch -x -D "cn=Manager,dc=yuikuen,dc=top" -w "Admin@123" -b "dc=yuikuen,dc=top"
# extended LDIF
#
# LDAPv3
# base <dc=yuikuen,dc=top> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# yuikuen.top
dn: dc=yuikuen,dc=top
o: yuikuen top
dc: yuikuen
objectClass: top
objectClass: dcObject
objectClass: organization

# Manager, yuikuen.top
dn: cn=Manager,dc=yuikuen,dc=top
cn: Manager
objectClass: organizationalRole
description: LDAP Manager

# People, yuikuen.top
dn: ou=People,dc=yuikuen,dc=top
ou: People
objectClass: top
objectClass: organizationalUnit

# Group, yuikuen.top
dn: ou=Group,dc=yuikuen,dc=top
ou: Group
objectClass: top
objectClass: organizationalUnit

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 4
```