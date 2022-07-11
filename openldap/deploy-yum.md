# OpenLDAP(yum)部署

## 前置环境
1. 演示环境，直接关闭 SELinux 和 Firewalld

```bash
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装程序
1. 执行`yum`命令进行下载安装

```bash
$ yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel migrationtools

# 查看版本
$ slapd -VV
@(#) $OpenLDAP: slapd 2.4.44 (Aug 31 2021 14:48:49) $
        mockbuild@x86-02.bsys.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd
```

2. 生成管理员密码

```bash
$ slappasswd -s Admin@123
{SSHA}GJEm8DDGliPDM+/ip/lk5d+KDBGjR9TU
```

3. 修改配置文件，添加之前生成的密码

> *从 OpenLDAP-2.4.23 版本开始所有配置数据都保存在 `/etc/openldap/slapd.d/` 中，已无slapd.conf 作为配置文件使用了*

```bash
$ vim /etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif
# 修改成自己的域名，管理员dn账号
olcSuffix: dc=yuikuen,dc=top
olcRootDN: cn=Manager,dc=yuikuen,dc=top
# 添加之前生成的密码
olcRootPW: {SSHA}GJEm8DDGliPDM+/ip/lk5d+KDBGjR9TU
```
```bash
$ vim /etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif
# 修改域名信息
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=extern
al,cn=auth" read by dn.base="cn=Manager,dc=yuikuen,dc=top" read by * none
```

4. 验证配置文件是否正确

```bash
$ slaptest -u
62c25987 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif"
62c25987 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif"
config file testing succeeded
```
注：如出现 `checksum error on`，可忽略错误继续操作

5. 启动服务并设置开机自启

```bash
$ systemctl enable --now slapd && systemctl status slapd
$ netstat -anpl|grep 389
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      5488/slapd          
tcp6       0      0 :::389                  :::*                    LISTEN      5488/slapd
```

## 配置程序

> 安装 openldap 后，会有三个命令用于修改配置文件，分别为 ldapadd，ldapmodify，ldapdelete，顾名思义就是添加，修改和删除。需要修改或增加配置时，则需要先写一个 ldif 后缀的配置文件，然后通过命令将写的配置更新到 slapd.d 目录下的配置文件中去

1. 配置数据库(OpenLDAP 默认使用的数据库是 BerkeleyDB)

```bash
# 程序目录内有example模版，可直接复制使用
$ cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
# 安装程序时会自动创建ldap用户，直接赋权
$ chown ldap:ldap -R /var/lib/ldap
$ chmod 700 -R /var/lib/ldap
```
注：`/var/lib/ldap` 为 BerkeleyDB 数据库默认存储的路径

2. 导入基本 `Schema` 文件

> schema 控制着条目有哪些对象类和属性，可自行选择导入

```bash
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/collective.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/corba.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/duaconf.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/dyngroup.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/java.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/misc.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/pmi.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/ppolicy.ldif
```

3. 修改 `migrate_common.ph` 文件，此文件主要用于生成 `ldif` 文件

```bash
$ vim /usr/share/migrationtools/migrate_common.ph +71
# 修改域名信息
$DEFAULT_MAIL_DOMAIN = "yuikuen.top";
$DEFAULT_BASE = "dc=yuikuen,dc=top";
$EXTENDED_SCHEMA = 1;
```

4. 重启服务以保证配置文件生效

```bash
$ systemctl restart slapd
```

## 创建组织

> 上面其实已经安装好 OpenLDAP 了，但里面什么都没有，现需要在此基础上，创建一个组织(公司 company)
> 以 `yuikuen.top` 为域名，并在其下创建 `Manager` 的组织角色(此用户为管理员，具有整个LDAP的权限)
> 之后再创建两个组织单元 `People/Group`，注意此处的组织单元并不是部门的意思。

1. 创建一个组织

```bash
$ mkdir -p /etc/openldap/myldif
$ vim ./myldif/base.ldif
# 此处有坑！！注意ldif文件都是以空行作为用户分割，格式要保持一致！！
dn: dc=yuikuen,dc=top
o: yuikuen top
dc: yuikuen
objectClass: top
objectClass: dcObject
objectclass: organization

dn: cn=Manager,dc=yuikuen,dc=top
cn: Manager
objectClass: organizationalRole
description: Directory Manager

dn: ou=People,dc=yuikuen,dc=top
ou: People
objectClass: top
objectClass: organizationalUnit

dn: ou=Group,dc=yuikuen,dc=top
ou: Group
objectClass: top
objectClass: organizationalUnit

# 此处为ACL控制，仅Manager可修改密码，不允许匿名访问(可略过，自行选择)
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by dn="cn=Manager,dc=yuikuen,dc=top" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=Manager,dc=yuikuen,dc=top" write by * read
```

2. 执行 `ldapadd` 命令创建组织

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f /etc/openldap/myldif/base.ldif
# 注：此处必须有输出如下adding信息才算创建成功
adding new entry "dc=yuikuen,dc=top"
adding new entry "cn=Manager,dc=yuikuen,dc=top"
adding new entry "ou=People,dc=yuikuen,dc=top"
adding new entry "ou=Group,dc=yuikuen,dc=top"
```

3. 添加组织架构

> 刚也说过 `People/Group` 并不是部门，只是组织单元。而现创建的才是部门或组
> 现以IT部门为例，创建各组别 op运维、rd开发、pm项目、qa测试

```bash
$ vim ./myldif/group.ldif
# 此处有坑！！注意ldif文件都是以空行作为用户分割，格式要保持一致！！
dn: ou=op,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: organizationalUnit
ou: op

dn: ou=rd,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: organizationalUnit
ou: rd

dn: ou=pm,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: organizationalUnit
ou: pm

dn: ou=qa,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: organizationalUnit
ou: qa
```

4. 执行创建命令并检查配置是否正确

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f /etc/openldap/myldif/group.ldif
# 注：此处必须有输出如下adding信息才算成功创建
adding new entry "ou=op,ou=People,dc=yuikuen,dc=top"
adding new entry "ou=rd,ou=People,dc=yuikuen,dc=top"
adding new entry "ou=pm,ou=People,dc=yuikuen,dc=top"
adding new entry "ou=qa,ou=People,dc=yuikuen,dc=top"
```
检查输出的配置内容，必须要有 `Success` 信息才算成功
```bash
$ ldapsearch -x -D cn=Manager,dc=yuikuen,dc=top -w Admin@123 -b "dc=yuikuen,dc=top"
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
description: Directory Manager

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

# op, People, yuikuen.top
dn: ou=op,ou=People,dc=yuikuen,dc=top
objectClass: organizationalUnit
ou: op

# rd, People, yuikuen.top
dn: ou=rd,ou=People,dc=yuikuen,dc=top
objectClass: organizationalUnit
ou: rd

# pm, People, yuikuen.top
dn: ou=pm,ou=People,dc=yuikuen,dc=top
objectClass: organizationalUnit
ou: pm

# qa, People, yuikuen.top
dn: ou=qa,ou=People,dc=yuikuen,dc=top
objectClass: organizationalUnit
ou: qa

# search result
search: 2
result: 0 Success

# numResponses: 9
# numEntries: 8
```
**参数说明：**
- -X：启动认证
- -D：bind admin 的 dn
- -w：admin 的密码
- -b：basedn，查询的基础 dn

## 安装终端

> 此处为什么要安装终端(PhpLDAPAdmin)？其实经过以上安装及配置，OpenLDAP 已经可以使用了。
> 只是现服务下默认没有普通用户，只有刚创建好的管理员用户和基础组织架构。而 `PhpLDAPadmin` 只是个图形管理工具，
> 因为 LDAP 的命令行管理方式对一般人来说并不是很友好，为了更好的演示用户和组的创建过程，在此使用 `PhpLDAPAdmin` 图形管理工具来进行，也可使用 `LDAP admin`。

1. 安装依赖包，并下载安装 phpldapadmin

```bash
$ yum -y install httpd php php-ldap php-gd php-mbstring php-pear php-bcmath php-xml
$ yum -y install epel-release 
$ yum --enablerepo=epel -y install phpldapadmin
```

1111111111111111111
12222222222222222