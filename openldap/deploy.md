# 服务部署





















## 安装 OpenLDAP

1. 环境配置，关闭 SELinux 和 Firewalld

```bash
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

2. 执行 Yum 命令进行下载安装

```bash
$ yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel migrationtools

# 查看版本
$ slapd -VV
@(#) $OpenLDAP: slapd 2.4.44 (Aug 31 2021 14:48:49) $
        mockbuild@x86-02.bsys.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd
```

3. 生成管理员密码

```bash
$ slappasswd -s Admin@123
{SSHA}GJEm8DDGliPDM+/ip/lk5d+KDBGjR9TU
```

4. 修改配置文件，添加上生成的密码

   > *从 OpenLDAP-2.4.23 版本开始所有配置数据都保存在 `/etc/openldap/slapd.d/` 中，已无slapd.conf 作为配置文件使用了*

```bash
$ vim /etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif
# 修改自己的域名和添加刚生成的管理员密码
olcSuffix: dc=yuikuen,dc=top
olcRootDN: cn=Manager,dc=yuikuen,dc=top
olcRootPW: {SSHA}GJEm8DDGliPDM+/ip/lk5d+KDBGjR9TU

$ vim /etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif
# 同上修改域名信息
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=extern
al,cn=auth" read by dn.base="cn=Manager,dc=yuikuen,dc=top" read by * none
```

```bash
# ACL控制(非必须)，只能自己可修改密码，不允许匿名访问，允许指定用户修改
$ vim acl.ldif
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by dn="cn=Manager,dc=yuikuen,dc=top" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=Manager,dc=yuikuen,dc=top" write by * read

$ ldapmodify -H ldapi://  -Y EXTERNAL -f addacl.sh
```

5. 验证配置文件是否正确

```bash
$ slaptest -u
62c25987 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif"
62c25987 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif"
config file testing succeeded
```

注：如出现 `checksum error on`，可忽略错误继续操作

6. 启动服务并设置开机自启

```bash
$ systemctl enable --now slapd && systemctl status slapd
$ netstat -anpl|grep 389
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      5488/slapd          
tcp6       0      0 :::389                  :::*                    LISTEN      5488/slapd
```

## 配置 OpenLDAP

> 安装 openldap 后，会有三个命令用于修改配置文件，分别为 ldapadd，ldapmodify，ldapdelete，顾名思义就是添加，修改和删除。需要修改或增加配置时，则需要先写一个ldif后缀的配置文件，然后通过命令将写的配置更新到 slapd.d 目录下的配置文件中去

1. 配置数据库，OpenLDAP 默认使用的数据库是 BerkeleyDB

```bash
$ cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
$ chown ldap:ldap -R /var/lib/ldap
$ chmod 700 -R /var/lib/ldap
```

注：`/var/lib/ldap` 为 BerkeleyDB 数据库默认存储的路径

2. 导入基本 Schema 文件

   > schema 控制着条目有哪些对象类和属性，可自行选择导入

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/collective.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/corba.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/duaconf.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/dyngroup.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/java.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/misc.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/pmi.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/ppolicy.ldif
```

3. 修改 `migrate_common.ph` 文件，主要用于生成 ldif 文件

```bash
$ vim /usr/share/migrationtools/migrate_common.ph +71
$DEFAULT_MAIL_DOMAIN = "yuikuen.top";
$DEFAULT_BASE = "dc=yuikuen,dc=top";
$EXTENDED_SCHEMA = 1;
```

4. 重启服务以保证配置文件生效

```bash
$ systemctl restart slapd
```

5. 创建一个组织

   > 在上述基础上，创建一个 company 的组织，yuikuen 为域名，并在其下创建 Manager 的组织角色(该组织角色内的用户具有整个 LDAP 的权限和 People/Group 两个组织单元)

```bash
$ vim mybase.ldif
# 此处有坑！！注意ldif文件以空行作为用户分割，格式要保持一致！！
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

$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f mybase.ldif
# 注：此处必须有输出adding信息才算成功创建
adding new entry "dc=yuikuen,dc=top"

adding new entry "cn=Manager,dc=yuikuen,dc=top"

adding new entry "ou=People,dc=yuikuen,dc=top"

adding new entry "ou=Group,dc=yuikuen,dc=top"
```
以下为配置解析及参考值(非必须)
```bash
# 创建一个yuikuen的组织，域名为yuikuen
dn: dc=yuikuen,dc=top
o: yuikuen top
dc: yuikuen
objectClass: top
objectClass: dcObject
objectclass: organization

# 创建一个Manager的组织角色(具有管理整个LDAP的权限)
dn: cn=Manager,dc=yuikuen,dc=top
cn: Manager
objectClass: organizationalRole
description: Directory Manager

# 此处跟之前的配置有一个不一样，这里以部门组织方式，下面为组方式
# 创建一个组织单元dc=IT,dc=yuikuen,dc=top
dn: dc=IT,dc=yuikuen,dc=top
changetype: add
dc: IT
objectClass: top
objectClass: dcObject
objectClass: organization
o: IT

# 在组织单元IT下创建两个子属性，ou=People,Group
dn: ou=People,dc=yuikuen,dc=top
ou: People
objectClass: top
objectClass: organizationalUnit

dn: ou=Group,dc=yuikuen,dc=top
ou: Group
objectClass: top
objectClass: organizationalUnit

# 添加用户&组时，需要指向组织单元IT,如下：
dn: cn=rd001,ou=rd,ou=People,dc=IT,dc=yuikuen,dc=top
```

6. 添加企业组织架构

   > 以 IT 部为例，op 运维、rd 开发、pm 项目、qa 测试等创建 ou 组别

```bash
$ vim group.ldif
# 此处有坑！！注意ldif文件以空行作为用户分割，格式要保持一致！！
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

# 执行添加命令
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f group.ldif
# 注：此处必须有输出adding信息才算成功创建
adding new entry "ou=op,ou=People,dc=yuikuen,dc=top"

adding new entry "ou=rd,ou=People,dc=yuikuen,dc=top"

adding new entry "ou=pm,ou=People,dc=yuikuen,dc=top"

adding new entry "ou=qa,ou=People,dc=yuikuen,dc=top"

# 检查内容是否正确，注：此处必须有输出Success信息才算成功
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

## 安装 PHPLDAPadmin

> 以上安装配置好对应的 OpenLDAP 相关服务后，其实已经可以使用的了，但该服务下默认没有普通用户，只有刚创建的管理员用户，账号 `cn=Manager,dc=yuikuen,dc=top`/密码 Admin@123
>
> 而 PHPLDAPadmin 是个图形化管理工具，LDAP 的命令行管理工具对一般人来说并不是很友好，为了更好的演示用户和组的创建过程，在此使用 PHPLDAPadmin 图形化管理工具进行

1. 安装依赖和 phpldapadmin

```bash
$ yum -y install httpd php php-ldap php-gd php-mbstring php-pear php-bcmath php-xml
$ yum -y install epel-release 
$ yum --enablerepo=epel -y install phpldapadmin
```

2. 修改配置文件，放开外网访问

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

3. 修改 uid 登录方式和关闭匿名登录和用户属性的唯一性(非必须修改)

```bash
$ vim /etc/phpldapadmin/config.php
# 398行，默认为uid进行登录，修改成dn(全称)或cn(用户名)
# ！！此处有坑，提示密码或搭建后无法登录时，请检查登录方式！！
$servers->setValue('login','attr','dn');

# 460行，把true改为false 关闭匿名登录
$servers->setValue('login','anon_bind',false);

# 520行，设置用户属性的唯一性,这里将cn sn给添加上
$servers->setValue('unique','attrs',array('mail','uid','uidNumber','cn','sn'))
```

修改完成后启动服务

```bash
$ systemctl enable --now httpd && systemctl status httpd
```

注：通过上述步骤，已经设置好一个 LDAP 目录树，其基准 dc=yuikuen,dc=top 是该树的根节点，其下一个管理域 cn=Manager,dc=yuikuen,dc=top，和两个组织单元 ou=People,dc=yuikuen,dc=top 及 ou=Group,dc=yuikuen,dc=top，而 People 下有多个子属性(小组 rd/op/pm/qa) 

![image-20220704113600171](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704113600171.png)

## 添加 People

> 配置完成后就可以登录 http://ldap-server/phpldapadmin，默认只有之前创建的管理员和组织&组织架构
>
> 账号 `cn=Manager,dc=yuikuen,dc=top` /密码 `Admin@123`

1. 创建用户，使用 sha1 存储密码，通过 userPassword 保存密码

```bash
$ slappasswd -h {sha} -s 123456
{SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
```

2. 创建新用户的 ldif 文件

```bash
$ vim user.ldif
# 此处有坑！！注意ldif文件以空行作为用户分割，格式要保持一致！！
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

3. 执行创建命令并查询其中一个用户是否成功

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f user.ldif
# 跟之前一样，有正常输出adding成功信息为成功
adding new entry "cn=rd001,ou=rd,ou=People,dc=yuikuen,dc=top"

adding new entry "cn=op001,ou=op,ou=People,dc=yuikuen,dc=top"

adding new entry "cn=pm001,ou=pm,ou=People,dc=yuikuen,dc=top"

adding new entry "cn=qa001,ou=qa,ou=People,dc=yuikuen,dc=top"

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

在配置第三方认证的时候，就是通过这样 userfilter 来 search 用户的，在 phpLDAPadmin 点击刷新就可以查看创建成功的用户

![image-20220704114009136](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704114009136.png)

**小知识：**命令行增、删、改属性，非管理员使用并不友好

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

## 添加 memberof 模块

> OpenLDAP 加载 memberof 模块后，可通过 groupOfNames 和 memberOf 实现分组认证的功能，在 Gitlab&Jenkins 等系统上都能作过滤组用户来使用，建议配置上。

1. 创建 module_group.sh 

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

2. 创建 group_objectClass.sh

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

## 添加 Group

> ou 并不能当做分组，只是组织架构的一个单元，ldap 的分组都是通过单独的 group 来实现的，分组类型如下：
>
> - groupOfNames：适用大多数用途，本次演示使用就是此
> - posixGroup：代表传统 unix 组，由 gidNUmber 和列表 memberUid 标识

注：OpenLDAP 用户和用户组之间是没有任何关联的，需要将新建或添加用户到用户组的 ldif 文件

1. 添加 Group 组织下的分组，并将用户绑定

```bash
$ vim addgroup.ldif
# 此处的cn=rd为在group组织下的组名，member是将cn=rd001添加此组下
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

$ ldapmodify -a -H ldap://188.188.4.140:389 -D "cn=Manager,dc=yuikuen,dc=top" -w Admin@123 -f addgroup.ldif
adding new entry "cn=rd,ou=Group,dc=yuikuen,dc=top"

adding new entry "cn=op,ou=Group,dc=yuikuen,dc=top"

adding new entry "cn=pm,ou=Group,dc=yuikuen,dc=top"

adding new entry "cn=qa,ou=Group,dc=yuikuen,dc=top"
```

此时可以看到之前创建 `cn=rd001,ou=rd,ou=People,dc=yuikuen,dc=top` 已创建好的 `cn=rd,ou=Group,dc=yuikuen,dc=top` 的 member 属性中

![image-20220704114738101](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704114738101.png)

2. 添加其他用户至指定 group 中(演示：现李四需要作测试的工作，将 op001 加到 cn=qa )

```bash
$ vim newaddgroup.ldif
dn: cn=qa,ou=Group,dc=yuikuen,dc=top
changetype: modify
add: member
member: cn=op001,ou=qa,ou=People,dc=yuikuen,dc=top

$ ldapmodify -a -H ldap://188.188.4.44:389 -D "cn=Manager,dc=yuikuen,dc=top" -w Admin@123 -f newaddgroup.ldif
```

3. 从 group 下移除 user(演示：现赵六离职，需要从组内移除此人)

```bash
$ vim removeuser.sh
dn: cn=qa,ou=Group,dc=yuikuen,dc=top
changetype: modify
delete: member
member: cn=qa001,ou=qa,ou=People,dc=yuikuen,dc=top

$ ldapmodify -H ldap:/// -x -D cn=Manager,dc=yuikuen,dc=top -w Admin@123 -f removeuser.sh
```

**参考链接**

- [运维干货: OpenLDAP搭建和使用 (qq.com)](https://mp.weixin.qq.com/s/zQVunrm9-oQWMnCeMxs9MQ)
- [Openldap安装部署 - 牧梦者 - 博客园 (cnblogs.com)](https://www.cnblogs.com/swordfall/p/12119010.html#auto_id_9)
231232132132131