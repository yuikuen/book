# OpenLDAP(yum)单节点安装2.4.x

> 随着公司服务器的增多，服务账户管理成为了瓶颈，过多的普通用户和管理用户，给运维工作带来了极大的不便，在此着手把账户管理部署起来。

## 前置环境

**程序版本**

- OpenLDAP：openldap-2.4.44-25.el7_9.x86_64
- Phpldapadmin：phpldapadmin-1.2.5-1.el7.noarch

1)演示环境，直接关闭 SELinux 和 Firewalld

```bash
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装服务

1)通过 yum 安装 OpenLDAP Server 服务

```bash
$ yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel
```

2)初始化配置，使用程序默认的模板建立数据库

```bash
$ cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
$ chown ldap:ldap -R /var/lib/ldap
$ chmod 700 -R /var/lib/ldap
```

注：`/var/lib/ldap` 为 BerkeleyDB 数据库默认存储的路径

3)启动服务并确认安装是否成功

```bash
$ systemctl enable --now slapd && systemctl status slapd
$ netstat -anpl|grep 389
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      1634/slapd          
tcp6       0      0 :::389                  :::*                    LISTEN      1634/slapd 
```

## 服务配置

> 上述操作只是基本的安装配置，用户、组织、设置等都是没有的，下面将进行基础配置及用户创建。

**这里有三个点需要注意的：**

1. 从 OpenLDAP-2.4.23 版本开始所有配置数据都保存在 `/etc/openldap/slapd.d/` 中，已无 slapd.conf 作为配置文件使用了
2. 安装 OpenLDAP 后，会有三个命令用于修改配置文件，分别为 `ldapadd`、`ldapmodify`、`ldapdelete`，顾名思义就是添加、修改和删除。需要修改或新增时，则需写一个 ldif 后缀的配置文件，然后通过命令将写的配置更新至 slapd.d 目录下的配置文件中
3. 最重要的事项，ldif文件都是以空行作为用户分割，格式要保持一致！！请参考 base.ldif 文件，勿加注释或空格之类！！

1)首先生成管理员的加密密码

```bash
$ slappasswd -s Admin@123
{SSHA}+IqqdRVpbUXCbre3dTrtGKt+zZKak9Ef
```

2)导入 Schema 基本文件(可根据需要添加更多)

> Schema 类似数据库表，定义了字段名和类型等参数值

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

3)修改域名信息(前面已说明过了，官方不推荐直接修改配置文件，建议通过 `ldapmodify` 来更新配置)

```bash
$ mkdir -p /etc/openldap/myldif
$ vim ./myldif/changedomain.ldif
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

**配置文件解释**

- 注意 ldif 文件都是以行作为用户分割，格式要保持一致
- 第一行：执行配置文件，到 `/etc/openldap/slapd.d/` 目录，找到指定 `cn=config/olcDatabase={0}config` 文件
- 第二行：changetype 指定类型为修改
- 第三行：add/replace 表示增加/替换的配置项
- 第四行：指定配置项的值

注：执行命令前，可以查看原来 `olcDatabase={0}config` 文件，里面是没 olcRootPW 配置项，执行后可以看到已新增刚指定的值

*另配置方法同理，直接修改配置文件*

```bash
$ vim /etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif
# 修改成自己的域名，管理员dn账号
olcSuffix: dc=yuikuen,dc=top
olcRootDN: cn=Manager,dc=yuikuen,dc=top
# 添加之前生成的密码
{SSHA}qDu/xSUWVnlMxDc/+EPJT0GB/x93p5Lk

$ vim /etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif
# 修改域名信息
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=extern
al,cn=auth" read by dn.base="cn=Manager,dc=yuikuen,dc=top" read by * none
```

4)执行命令进行修改

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

## 开启日志

> LDAP 默认是没有开启日志功能的，需要单独配置

1)配置 ldap_log

```bash
$ touch ./myldif/ldap_log.ldif; vim ldap_log.ldif
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats
```

2)执行 `ldapmodify` 命令发送参数到 LDAP 进行配置

```bash
$ ldapmodify -Y EXTERNAL -H ldapi:/// -f ldap_log.ldif
```

3)使用系统自带的 `rsyslog` 进行日志输出

```bash
$ vim /etc/rsyslog.conf
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

## 创建组织

> 通过上述操作后，OpenLDAP 的基础功能已实现了，但里面什么都没有。而现在就需要在此创建基础架构，创建一个组织(公司 Company)，以 `yuikuen.top` 域名，并在其下创建名为 `Manager` 的组织角色(管理员，具有整个 LDAP 的权限)，之后再创建两个组织单元 `People/Group`
>
> 注意：此处的组织单元并不是部门的意思，只是用户/部门组的集合单元

1)创建一个 ldif 文件，定义组织信息

```bash
$ vim ./myldif/base.ldif
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

2)执行命令创建组织

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f base.ldif
adding new entry "dc=yuikuen,dc=top"

adding new entry "cn=Manager,dc=yuikuen,dc=top"

adding new entry "ou=People,dc=yuikuen,dc=top"

adding new entry "ou=Group,dc=yuikuen,dc=top"
```

3)使用 `ldapsearch` 来检查内容

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

## 创建用户

> ou 并不能当做分组，而仅仅是组织架构的一个单元，ldap 的分组都是通过单独的 Group 来实现的

1)首先生成一个加密的密码(可后面自行修改)

```bash
$ slappasswd -h {sha} -s 123456
{SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
```

2)创建 ldif 文件，定义用户&用户组参数值

> 以 IT 部门为例：开发、运维、项目等级别创建，并创建一位开发同事

```bash
$ vim ./myldif/user.ldif
# 创建三个用户集合rd/op/pm
dn: ou=rd,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: organizationalUnit
ou: rd

dn: ou=op,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: organizationalUnit
ou: op

dn: ou=pm,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: organizationalUnit
ou: pm

# 创建一个rd组的用户rd001，并定义相关个人信息
dn: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
cn: rd001
uid: rd001
sn: rd001
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
displayName: 小王
title: Java_Engineer
mail: rd001@yuikuen.top
shadowMin: 0
ShadowMax: 99999
shadowWarning: 7
loginShell: /bin/bash
gecos: Demo [Demo user example]
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/rd001

# 创建一个部门组back-end，将uid=rd001的用户进行关联
dn: cn=back-end,ou=Group,dc=yuikuen,dc=top
objectClass: posixGroup
objectClass: top
cn: back-end
gidNumber: 100000
memberUid: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
```

3)执行命令进行创建

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f user.ldif 
adding new entry "ou=rd,ou=People,dc=yuikuen,dc=top"

adding new entry "ou=op,ou=People,dc=yuikuen,dc=top"

adding new entry "ou=pm,ou=People,dc=yuikuen,dc=top"

adding new entry "uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top"

adding new entry "cn=back-end,ou=Group,dc=yuikuen,dc=top"
```

4)指定唯一 id 来查询某个用户，比如 cn 唯一

```bash
# 示例：ldapsearch -x -w "admin_passwd" -D "admin_dn" -b "base_dn" "search_filter"
$ ldapsearch -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -b "dc=yuikuen,dc=top" "cn=rd001"
# extended LDIF
#
# LDAPv3
# base <dc=yuikuen,dc=top> with scope subtree
# filter: cn=rd001
# requesting: ALL
#

# rd001, rd, People, yuikuen.top
dn: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
cn: rd001
uid: rd001
sn: rd001
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
userPassword:: e1NIQX1mRXFOQ2NvM1lxOWg1WlVnbEQzQ1pKVDRsQnM9
displayName:: 5bCP546L
title: Java_Engineer
mail: rd001@yuikuen.top
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
loginShell: /bin/bash
gecos: Demo [Demo user example]
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/rd001

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

5)执行 `ldapdelete` 删除指定用户，再次查询已提示没有此用户

```bash
$ ldapdelete -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -x "uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top"
$ ldapsearch -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -b "dc=yuikuen,dc=top" "cn=rd001"
# extended LDIF
#
# LDAPv3
# base <dc=yuikuen,dc=top> with scope subtree
# filter: cn=rd001
# requesting: ALL
#

# search result
search: 2
result: 0 Success

# numResponses: 1
```

## 管理工具

> 此处为什么要安装管理工具？其实经过以上安装及配置，OpenLDAP 已经可以使用了。 但 LDAP 的命令行管理方式对一般人来说并不是很友好，为了更方便管理用户和组的创建，可以使用 `PhpLDAPAdmin` 或 `LDAP Admin` 图形管理工具来进行管理

### PhpLDAPAdmin

> 基于 Web 的 OpenLDAP 管理工具，通过代理实现

1)安装依赖包、服务组件，并下载 phpldapadmin

```bash
$ yum -y install httpd php php-ldap php-gd php-mbstring php-pear php-bcmath php-xml
$ yum -y install epel-release 
$ yum --enablerepo=epel -y install phpldapadmin
```

2)修改配置文件，开放外网访问权限

```bash
$ vim /etc/httpd/conf.d/phpldapadmin.conf
Alias /phpldapadmin /usr/share/phpldapadmin/htdocs
Alias /ldapadmin /usr/share/phpldapadmin/htdocs

<Directory /usr/share/phpldapadmin/htdocs>
  <IfModule mod_authz_core.c>
    # Apache 2.4
    # 将local改成all granted
    Require all granted
  </IfModule>
  <IfModule !mod_authz_core.c>
    # Apache 2.2
    Order Deny,Allow
    Deny from all
    Allow from 127.0.0.1
    Allow from ::1
  </IfModule>
</Directory>
```

3)修改 uid 登录方式和关闭匿名登录、用户属性的唯一性等

```bash
$ vim /etc/phpldapadmin/config.php +398
# 此处注意！为什么网上教程搭建后账密无法登录，因默认为uid登录方式
# dn为域名全称，cn仅为用户名
$servers->setValue('login','attr','dn');

# nu:460，将true改为false，关闭匿名登录
$servers->setValue('login','anon_bind',false);

# nu:520，设置用户属性的唯一，把cn,sn加上
$servers->setValue('unique','attrs',array('mail','uid','uidNumber','cn','sn'))
```

4)修改完成后启动服务

```bash
$ systemctl enable --now httpd && systemctl status httpd
```

5)配置完成，打开浏览器登录  http://ldap-server/phpldapadmin 

- 账号：`cn=Manager,dc=yuikuen,dc=top`
- 密码：`Admin@123`

![image-20220721161200623](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220721161200623.png)

### LDAP Admin

> Windows 下的 OpenLDAP 管理工具，直接解压打开 exe 文件即可使用

1)到 [LDAP Admin官网](http://www.ldapadmin.org/index.html)，找到 [Download](https://sourceforge.net/projects/ldapadmin/files/ldapadmin/1.8.3/) 下载（当前最新版本是 1.8.3）

![image-20220721161548513](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220721161548513.png)

![image-20220721161705076](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220721161705076.png)

## 参考链接

- [官方文档](https://www.openldap.org/doc/admin24/guide.html)
- [详细的ldap介绍](https://www.cnblogs.com/kevingrace/p/5773974.html)
- [详细的ldap安装介绍](https://www.cnblogs.com/kevingrace/p/9052669.html)
- [memberOf模块介绍](https://www.jianshu.com/p/c877b317f294)
- [运维干货: OpenLDAP搭建和使用 (qq.com)](https://mp.weixin.qq.com/s/zQVunrm9-oQWMnCeMxs9MQ)
- [Openldap安装部署 - 牧梦者 - 博客园 (cnblogs.com)](https://www.cnblogs.com/swordfall/p/12119010.html#auto_id_9)
- [CentOS7 配置OpenLDAP（一） 配置OpenLDAP服务单节点模式并实现服务器登录管理常见场景](https://blog.csdn.net/oyym_mv/article/details/94404663)