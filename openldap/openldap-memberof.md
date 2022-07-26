# OpenLDAP启用MemberOf属性

默认情况下 OpenLDAP 的用户组属性是 Posixgroup，而 Posixgroup 用户组和用户是没有实际的对应关系，如需要关联起来则需要将用户添加到对应组中，如之前创建的 `rd001`，通过 `memberUid` 来关联。
那之前创建的很多 ou(rd、op、pm)，不就是 Group 吗？理论上是 Group，但 ou 的 objectClass 并没设置成 Group。ldap 的 Group 是一种单独的类型 `objectClass: groupOfNames`，字段名叫做 `member`，value 就是 entry 的 dn，如此实现了 group & user 的映射关系。
很多场景下，需要快速指定用户是属于哪一个或多个组(memberof)，如常用的 GitLab、Jenkins、Harbor 集成 OpenLDAP 的时候，指定 `groupOfUniqueNames` 用户组中通过 `uniquemember` 属性新增一个用户，LDAP 就会自动在该用户上创建一个 `memberOf` 属性，其值为该组的 dn

## 注册模块

1)不同发行版本，模块路径可能不同，CentOS7 rpm安装(yum)的模块路径为 `/usr/lib64/openldap`

```bash
$ find / -name memberof.la

$ ll /etc/openldap/slapd.d/cn\=config
total 24
drwxr-x--- 2 ldap ldap 4096 Jul 21 15:00 cn=schema
-rw------- 1 ldap ldap  378 Jul 21 14:59 cn=schema.ldif
-rw------- 1 ldap ldap  513 Jul 21 14:59 olcDatabase={0}config.ldif
-rw------- 1 ldap ldap  443 Jul 21 14:59 olcDatabase={-1}frontend.ldif
-rw------- 1 ldap ldap  606 Jul 21 15:01 olcDatabase={1}monitor.ldif
-rw------- 1 ldap ldap  966 Jul 21 15:01 olcDatabase={2}hdb.ldif
```

2)创建一个文件，加载 memberof 模块使用

```bash
$ touch ./cn=config/memberof.ldif; vim memberof.ldif
dn: cn=module{0},cn=config
cn: module{0}
objectClass: olcModuleList
objectclass: top
olcModuleload: memberof.la
olcModulePath: /usr/lib64/openldap

dn: olcOverlay={0}memberof,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfUniqueNames
olcMemberOfMemberAD: uniqueMember
olcMemberOfMemberOfAD: memberOf
```
注：老的版本是 `groupOfNames,member`. 新版本是 `groupOfUniqueNames,uniqueMember`

3)执行 `ldapadd` 加载模块

```bash
$ ldapadd -Q -Y EXTERNAL -H ldapi:/// -f memberof.ldif
adding new entry "cn=module{0},cn=config"

adding new entry "olcOverlay={0}memberof,olcDatabase={2}hdb,cn=config"

# 再查看目录，会生成cn=module{0}.ldif文件
$ ll /etc/openldap/slapd.d/cn\=config
total 32
-rw------- 1 ldap ldap  554 Jul 21 17:02 cn=module{0}.ldif
drwxr-x--- 2 ldap ldap 4096 Jul 21 15:00 cn=schema
-rw------- 1 ldap ldap  378 Jul 21 14:59 cn=schema.ldif
-rw-r--r-- 1 root root  477 Jul 21 17:02 memberof.ldif
-rw------- 1 ldap ldap  513 Jul 21 14:59 olcDatabase={0}config.ldif
-rw------- 1 ldap ldap  443 Jul 21 14:59 olcDatabase={-1}frontend.ldif
-rw------- 1 ldap ldap  606 Jul 21 15:01 olcDatabase={1}monitor.ldif
drwxr-x--- 2 ldap ldap   41 Jul 21 17:02 olcDatabase={2}hdb
-rw------- 1 ldap ldap  966 Jul 21 15:01 olcDatabase={2}hdb.ldif
```

## 加载 refint

1)创建一个 ldif 文件，加载 refint

```bash
$ touch reint1.ldif; vim refint1.ldif
dn: cn=module{0},cn=config
add: olcmoduleload
olcmoduleload: refint
```
注：此处的{编号}是根据上一步骤创建的文件名

2)执行 `ldapmodify` 加载

```bash
$ ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f refint1.ldif
modifying entry "cn=module{0},cn=config"
```

3)再创建一个 ldif 文件，添加 Schema

```bash
$ touch refint2.ldif; vim refint2.ldif
dn: olcOverlay=refint,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: refint
olcRefintAttribute: memberof uniqueMember manager owner
```

4)执行 `ldapadd` 进行添加

```bash
$ ldapadd -Q -Y EXTERNAL -H ldapi:/// -f refint2.ldif
adding new entry "olcOverlay=refint,olcDatabase={2}hdb,cn=config"
```

## 验证模块

1)查看当前 dn 下包含 cn=config 配置列表是否显示已生效的 memberof 和 refint 配置

```bash
$ ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config dn   |  grep olcOverlay
dn: olcOverlay={0}memberof,olcDatabase={2}hdb,cn=config
dn: olcOverlay={1}refint,olcDatabase={2}hdb,cn=config
```

2)添加用户和组

```bash
$ vim add_user.ldif
dn: uid=pm001,ou=pm,ou=People,dc=yuikuen,dc=top
cn: pm001
uid: pm001
sn: pm001
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
displayName: 熊工
title: Project_Engineer
mail: pm001@yuikuen.top
shadowMin: 0
ShadowMax: 99999
shadowWarning: 7
loginShell: /bin/bash
gecos: Demo [Demo user example]
uidNumber: 20001
gidNumber: 20001
homeDirectory: /home/pm001

dn: cn=gerrit,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfUniqueNames
cn: gerrit
uniqueMember: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
uniqueMember: uid=pm001,ou=pm,ou=People,dc=yuikuen,dc=top
```

3)执行命令并查看验证

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f add_user.ldif 
adding new entry "uid=pm001,ou=pm,ou=People,dc=yuikuen,dc=top"

adding new entry "cn=gerrit,ou=Group,dc=yuikuen,dc=top"
```

```bash
$ ldapsearch -LL -Y EXTERNAL -H ldapi:///  "(cn=gerrit)" -b dc=yuikuen,dc=top
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
version: 1

dn: cn=gerrit,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfUniqueNames
cn: gerrit
uniqueMember: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
uniqueMember: uid=pm001,ou=pm,ou=People,dc=yuikuen,dc=topldapsearch 

$ ldapsearch -LL -Y EXTERNAL -H ldapi:///  "(uid=pm001)" -b dc=yuikuen,dc=top memberof
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
version: 1

dn: uid=pm001,ou=pm,ou=People,dc=yuikuen,dc=top
memberOf: cn=gerrit,ou=Group,dc=yuikuen,dc=top
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220721175548.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220721175741.png)