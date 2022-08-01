# OpenLDAP User

> 添加用户和用户组的方式有两种，本篇主要讲解自定义方式。
> 
> - 将系统用户通过 `migrationtools` 工具生成 ldif 文件导入 LDAP 目录树中生成用户
> - 通过自定义 ldif 文件并通过 LDAP 命令进行添加或修改操作

## 创建用户

1）首先生成一个加密的密码

```bash
$ slappasswd -h {sha} -s 123456
{SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
```

2）创建 ldif 文件，定义用户集合组

> 注意：此处的用户组指的是 People 组织单元下的用户集合，比如运维组、开发组、项目组等部门组别

```bash
$ cat add_team.ldif
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
```
```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f add_team.ldif
adding new entry "ou=rd,ou=People,dc=yuikuen,dc=top"

adding new entry "ou=op,ou=People,dc=yuikuen,dc=top"

adding new entry "ou=pm,ou=People,dc=yuikuen,dc=top"
```

3）再创建 ldif 文件，定义用户及相关信息参数

```bash
$ cat add_user.ldif
dn: uid=op001,ou=op,ou=People,dc=yuikuen,dc=top
uid: op001
cn: op001
sn: op001
givenName: yuen
displayName: yuikuen
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
uidNumber: 1000
gidNumber: 1000
gecos: System Manager
loginShell: /bin/bash
homeDirectory: /home/ldapusers
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs= 
shadowLastChange: 17654
shadowMin: 0
shadowMax: 999999
shadowWarning: 7
shadowExpire: -1
employeeNumber: 10001
homePhone: 0769-xxxxxxxx
mobile: 181xxxxxxxx
mail: yuikuen.yuen@hotmail.com
postalAddress: DongGuan
initials: Sys_Engineer
```
```bash
# example 说明
dn: uid=op001,ou=op,ou=People,dc=yuikuen,dc=top
uid: op001                                         // OpenLDAP
cn: op001
sn: op001
givenName: yuen
displayName: yuikuen
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
uidNumber: 1000                                    // 账号的UID
gidNumber: 1000                                    // 账号的GID 
gecos: System Manager                              // 登录欢迎词
loginShell: /bin/bash                              // 用户登入的SHELL
homeDirectory: /home/ldapusers                     // 用户的主目录指定
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=    // 加密密码
shadowLastChange: 17654                            // 用户最后一次修改的密码时间
shadowMin: 0                                       // 允许密码修改的天数(0代表任何)
shadowMax: 999999                                  // 系统强制用户修改新密码的天数
shadowWarning: 7                                   // 密码过期的保留天数(7天)
shadowExpire: -1                                   // 密码过期后的账号状态
employeeNumber: 10001                              // 工号
homePhone: 0769-xxxxxxxx                           // 家庭电话
mobile: 181xxxxxxxx                                // 手机号码
mail: yuikuen.yuen@hotmail.com                     // 电子邮箱
postalAddress: DongGuan                            // 住址信息
initials: Sys_Engineer                             // 职务
```

4）执行创建命令，再通过唯一 id 来查询该用户是否成功创建

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f add_user.ldif 
adding new entry "uid=op001,ou=op,ou=People,dc=yuikuen,dc=top"

$ ldapsearch -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -b "dc=yuikuen,dc=top" "uid=op001"
# extended LDIF
#
# LDAPv3
# base <dc=yuikuen,dc=top> with scope subtree
# filter: uid=op001
# requesting: ALL
#

# op001, op, People, yuikuen.top
dn: uid=op001,ou=op,ou=People,dc=yuikuen,dc=top
uid: op001
cn: op001
sn: op001
givenName: yuen
displayName: yuikuen
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
uidNumber: 1000
gidNumber: 1000
gecos: System Manager
loginShell: /bin/bash
homeDirectory: /home/ldapusers
userPassword:: e1NIQX1mRXFOQ2NvM1lxOWg1WlVnbEQzQ1pKVDRsQnM9IA==
shadowLastChange: 17654
shadowMin: 0
shadowMax: 999999
shadowWarning: 7
shadowExpire: -1
employeeNumber: 10001
homePhone: 0769-xxxxxxxx
mobile: 181xxxxxxxx
mail: yuikuen.yuen@hotmail.com
postalAddress: DongGuan
initials: Sys_Engineer

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

## 创建用户组

> **默认情况下 LDAP 的用户组属性是 `Posixgroup`，用户组和用户没有实际的对应关系，如需要关联起来则需要将用户添加到对应的组中。**
> 
> 注意：此处的用户组类似功能组，与之前的用户部门组不一样，举个栗子：
> 
> 开发组有3个用户，但只有开发组长才有访问某个项目组的权限，那我们就需要创建此项目组并添加开发组长来区分

1）创建一个 `general` 组，再创建一个用户，添将此用户添加至此组中

```bash
$ cat add_group.ldif
dn: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
uid: rd001
cn: rd001
sn: rd001
givenName: zhang
displayName: san
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
uidNumber: 2000
gidNumber: 2000
gecos: General User
loginShell: /bin/bash
homeDirectory: /home/ldapusers
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
shadowLastChange: 17654
shadowMin: 0
shadowMax: 999999
shadowWarning: 7
shadowExpire: -1
employeeNumber: 20001
homePhone: 0769-xxxxxxxx
mobile: 181xxxxxxxx
mail: rd001@yuikuen.top
postalAddress: DongGuan
initials: Java_Engineer

dn: cn=general,ou=Group,dc=yuikuen,dc=top
objectClass: posixGroup
objectClass: top
cn: general
gidNumber: 100000
memberUid: rd001
```

2）执行创建命令，并检查是否添加成功

```bash
ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f add_group.ldif 
adding new entry "uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top"

adding new entry "cn=general,ou=Group,dc=yuikuen,dc=top"

$ ldapsearch -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -b "dc=yuikuen,dc=top" "cn=general"
# extended LDIF
#
# LDAPv3
# base <dc=yuikuen,dc=top> with scope subtree
# filter: cn=general
# requesting: ALL
#

# general, Group, yuikuen.top
dn: cn=general,ou=Group,dc=yuikuen,dc=top
objectClass: posixGroup
objectClass: top
cn: general
gidNumber: 100000
memberUid: rd001

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220729102214.png)