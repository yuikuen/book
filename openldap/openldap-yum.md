# OpenLDAP(yum)安装2.4.x

**程序版本：**
- OpenLDAP：openldap-2.4.44-25.el7_9.x86_64
- Phpldapadmin：phpldapadmin-1.2.5-1.el7.noarch

## 前置环境
1)演示环境，直接关闭 SELinux 和 Firewalld

```bash
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装程序
1)执行`yum`命令进行下载安装

```bash
$ yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel migrationtools

# 查看版本
$ slapd -VV
@(#) $OpenLDAP: slapd 2.4.44 (Aug 31 2021 14:48:49) $
        mockbuild@x86-02.bsys.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd
```

2)生成管理员密码

```bash
$ slappasswd -s Admin@123
{SSHA}GJEm8DDGliPDM+/ip/lk5d+KDBGjR9TU
```

3)修改配置文件，添加之前生成的密码

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

4)验证配置文件是否正确

```bash
$ slaptest -u
62c25987 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif"
62c25987 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif"
config file testing succeeded
```
注：如出现 `checksum error on`，可忽略错误继续操作

5)启动服务并设置开机自启

```bash
$ systemctl enable --now slapd && systemctl status slapd
$ netstat -anpl|grep 389
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      5488/slapd          
tcp6       0      0 :::389                  :::*                    LISTEN      5488/slapd
```

## 配置程序

> 安装 openldap 后，会有三个命令用于修改配置文件，分别为 ldapadd，ldapmodify，ldapdelete，顾名思义就是添加，修改和删除。需要修改或增加配置时，则需要先写一个 ldif 后缀的配置文件，然后通过命令将写的配置更新到 slapd.d 目录下的配置文件中去

1)配置数据库(OpenLDAP 默认使用的数据库是 BerkeleyDB)

```bash
# 程序目录内有example模版，可直接复制使用
$ cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
# 安装程序时会自动创建ldap用户，直接赋权
$ chown ldap:ldap -R /var/lib/ldap
$ chmod 700 -R /var/lib/ldap
```
注：`/var/lib/ldap` 为 BerkeleyDB 数据库默认存储的路径

2)导入基本 `Schema` 文件

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

3)修改 `migrate_common.ph` 文件，此文件主要用于生成 `ldif` 文件

```bash
$ vim /usr/share/migrationtools/migrate_common.ph +71
# 修改域名信息
$DEFAULT_MAIL_DOMAIN = "yuikuen.top";
$DEFAULT_BASE = "dc=yuikuen,dc=top";
$EXTENDED_SCHEMA = 1;
```

4)重启服务以保证配置文件生效

```bash
$ systemctl restart slapd
```

## 创建组织

> 上面其实已经安装好 OpenLDAP 了，但里面什么都没有，现需要在此基础上，创建一个组织(公司 company)
> 以 `yuikuen.top` 为域名，并在其下创建 `Manager` 的组织角色(此用户为管理员，具有整个LDAP的权限)
> 之后再创建两个组织单元 `People/Group`，注意此处的组织单元并不是部门的意思。

1)创建一个组织

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

2)执行 `ldapadd` 命令创建组织

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f /etc/openldap/myldif/base.ldif
# 注：此处必须有输出如下adding信息才算创建成功
adding new entry "dc=yuikuen,dc=top"
adding new entry "cn=Manager,dc=yuikuen,dc=top"
adding new entry "ou=People,dc=yuikuen,dc=top"
adding new entry "ou=Group,dc=yuikuen,dc=top"
```

3)添加组织架构

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

4)执行创建命令并检查配置是否正确

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

1)安装依赖包，并下载安装 phpldapadmin

```bash
$ yum -y install httpd php php-ldap php-gd php-mbstring php-pear php-bcmath php-xml
$ yum -y install epel-release 
$ yum --enablerepo=epel -y install phpldapadmin
```

2)修改配置文件，放开外网访问权限

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

3)修改 uid 登录方式和关闭匿名登录、用户属性的唯一性等（非必须）

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

## 添加用户

> 配置完成后，可登录 http://ldap-server/phpldapadmin 查看
> 账密：`cn=Manager,dc=yuikuen,dc=top` / `Admin@123`

简要回顾：上述的操作是，设置一个 LDAP 目录树，其基准 `dc=yuikuen,dc=top` 是该树的根节点，
创建一个管理域为 `cn=Manager,dc=yuikuen,dc=top`，和两个组织单元 `ou=People,dc=yuikuen,dc=top` 及 
`ou=Group,dc=yuikuen,dc=top`，而 People 下拥有多个子属性(部门组 rd/op/pm/qa)

![image-20220704113600171](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704113600171.png)

如图所示，默认只有创建的管理员、组织和组织架构，而下面就开始创建所需的用户

1)首先第一步，创建用户，使用 sha1 存储密码，通过 userPassword 保存

```bash
$ slappasswd -h {sha} -s 123456
{SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
```

2)创建新用户的 ldif 文件

```bash
# 再次提醒！ldif文件以空行作用用户分割，格式要保持一致！
dn: cn=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: inetOrgPerson
cn: rd001
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
sn: rd001
title: Java_Engineer
mail: rd001@yuikuen.top
uid: rd001
displayName: 张三

dn: cn=op001,ou=op,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: inetOrgPerson
cn: op001
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
sn: op001
title: System_Engineer
mail: op001@yuikuen.top
uid: op001
displayName: 李四

dn: cn=pm001,ou=pm,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: inetOrgPerson
cn: pm001
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
sn: pm001
title: Project_Engineer
mail: pm001@yuikuen.top
uid: pm001
displayName: 王五

dn: cn=qa001,ou=qa,ou=People,dc=yuikuen,dc=top
changetype: add
objectClass: inetOrgPerson
cn: qa001
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
sn: qa001
title: Test_Engineer
mail: qa001@yuikuen.top
uid: qa001
displayName: 赵六
```

3)执行创建命令并查询其中一个用户测试是否成功

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f /etc/openldap/myldif/user.ldif
# 跟之前创建命令一样，要有正常输出adding信息才为成功
adding new entry "cn=rd001,ou=rd,ou=People,dc=yuikuen,dc=top"
adding new entry "cn=op001,ou=op,ou=People,dc=yuikuen,dc=top"
adding new entry "cn=pm001,ou=pm,ou=People,dc=yuikuen,dc=top"
adding new entry "cn=qa001,ou=qa,ou=People,dc=yuikuen,dc=top"

# 以cn=rd001为例进行验证
$ ldapsearch -x -D cn=Manager,dc=yuikuen,dc=top -w Admin@123 -b "dc=yuikuen,dc=top" "cn=rd001"
# extended LDIF
#
# LDAPv3
# base <dc=yuikuen,dc=top> with scope subtree
# filter: cn=rd001
# requesting: ALL
#

# rd001, rd, People, yuikuen.top
dn: cn=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
objectClass: inetOrgPerson
cn: rd001
userPassword:: e1NIQX1mRXFOQ2NvM1lxOWg1WlVnbEQzQ1pKVDRsQnM9
sn: rd001
title: Java_Engineer
mail: rd001@yuikuen.top
uid: rd001
displayName:: 5byg5LiJ

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```
注：在配置第三方认证的时候，就是通过 userfilter 来 search 用户的，在 PhpLDAPAdmin 上点击刷新，就可以查看到已创建成功的用户

![image-20220704114009136](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704114009136.png)

**小知识：**常用命令行操作指令
- 删除用户 & 删除组织

```bash
$ ldapdelete -x -D "cn=Manager,dc=yuikuen,dc=top" -w Admin@123 "uid=rd001,ou=People,dc=yuikuen,dc=top"
$ ldapdelete -x -D "cn=Manager,dc=yuikuen,dc=top" -w Admin@123 "ou=People,dc=yuikuen,dc=top"
```

- 管理员修改用户密码

```bash
$ vim changepwd.ldif
dn: cn=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
changetype: modify
# add是增加属性，而replace是修改已存在属性
replace: userPassword
userPassword: substring

# ldapmodify是用来修改密码的，执行命令
$ ldapmodify -a -H ldap://188.188.4.140:389 -D "cn=Manager,dc=yuikuen,dc=top" -w Admin@123 -f changepwd.ldif

# 默认没权限修改密码，在已知密码下修改密码
$ ldappasswd -h 188.188.4.140 -p 389 -x -D "cn=rd001,ou=rd，ou=People,dc=yuikuen,dc=top" -w substring -a old_passwd -S
```

## 添加模块

> OpenLDAP 加载 memberof 模块后，可通过 groupOfNames 和 memberOf 实现分组认证的功能，
> 在 Gitlab&Jenkins 等系统上都能作为过滤组用户来使用，为此在创建部门组前进行添加。

1)创建 `module_group.sh`

```bash
$ vim module_group.sh
dn: cn=module,cn=config
cn: module
objectClass: olcModuleList
olcModulePath: /usr/lib64/openldap

dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: memberof.la

$ ldapadd -Q -Y EXTERNAL -H ldapi:/// -f module_group.sh 
adding new entry "cn=module,cn=config"

modifying entry "cn=module{0},cn=config"
```

2)创建 `group_objectClass.sh`

```bash
$ vim group_objectClass.sh
dn: olcOverlay=memberof,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member     
olcMemberOfMemberOfAD: memberOf

$ ldapadd -Q -Y EXTERNAL -H ldapi:/// -f group_objectClass.sh
adding new entry "olcOverlay=memberof,olcDatabase={2}hdb,cn=config"
```

## 添加用户组

> ou 并不能当做分组，只是组织架构的一个单元，ldap 的分组都是通过单独的 group 来实现的，分组类型如下：
> - groupOfNames：适用大多数用途，本次演示类型
> - posixGroup：代表传统 unix 组，由 gidNUmber 和列表 memberUid 标识
> 注：OpenLDAP 用户和用户组之间默认是没有任何关联的，需要将新建或原用户，添加到指定用户组的 ldif 文件内

1)在 Group 单元组织下添加分组，并将用户绑定

```bash
$ vim addgroup.ldif
# 简要说明：将cn=rd001的用户，绑定group下新创建cn=rd的组下
dn: cn=rd,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfNames
cn: rd
member: cn=rd001,ou=rd,ou=People,dc=yuikuen,dc=top

dn: cn=op,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfNames
cn: op
member: cn=op001,ou=op,ou=People,dc=yuikuen,dc=top

dn: cn=pm,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfNames
cn: pm
member: cn=pm001,ou=pm,ou=People,dc=yuikuen,dc=top

dn: cn=qa,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfNames
cn: qa
member: cn=qa001,ou=qa,ou=People,dc=yuikuen,dc=top
```

2)执行命令进行创建绑定

```bash
$ ldapmodify -a -H ldap://188.188.4.140:389 -D "cn=Manager,dc=yuikuen,dc=top" -w Admin@123 -f addgroup.ldif
adding new entry "cn=rd,ou=Group,dc=yuikuen,dc=top"
adding new entry "cn=op,ou=Group,dc=yuikuen,dc=top"
adding new entry "cn=pm,ou=Group,dc=yuikuen,dc=top"
adding new entry "cn=qa,ou=Group,dc=yuikuen,dc=top"
```
此时可看到之前创建的用户，如 `cn=rd001` 已经添加到 `cn=rd,ou=Group,dc=yuikuen,dc=top` 的 member 属性中

![image-20220704114738101](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704114738101.png)

3)添加其他用户至指定组中（现李四需接手测试的工作，将 op001 添加到 cn=qa）

```bash
$ vim newaddgroup.ldif
dn: cn=qa,ou=Group,dc=yuikuen,dc=top
changetype: modify
add: member
member: cn=op001,ou=qa,ou=People,dc=yuikuen,dc=top

$ ldapmodify -a -H ldap://188.188.4.44:389 -D "cn=Manager,dc=yuikuen,dc=top" -w Admin@123 -f newaddgroup.ldif
```

4)从 group 下移除 user（现赵六离职，需从组内移除此人）

```bash
$ vim removeuser.sh
dn: cn=qa,ou=Group,dc=yuikuen,dc=top
changetype: modify
delete: member
member: cn=qa001,ou=qa,ou=People,dc=yuikuen,dc=top

$ ldapmodify -H ldap:/// -x -D cn=Manager,dc=yuikuen,dc=top -w Admin@123 -f removeuser.sh
```
注：通过以上操作，基本实现了 LDAP 用户&用户组管理

**参考链接：**
- [运维干货: OpenLDAP搭建和使用 (qq.com)](https://mp.weixin.qq.com/s/zQVunrm9-oQWMnCeMxs9MQ)
- [Openldap安装部署 - 牧梦者 - 博客园 (cnblogs.com)](https://www.cnblogs.com/swordfall/p/12119010.html#auto_id_9)