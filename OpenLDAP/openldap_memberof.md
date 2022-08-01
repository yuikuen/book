# OpenLDAP MemberOf

> 很多实际场景，需要快速指定用户是属于哪一个或多个组，如常用的 GitLab、Jenkins、Harbor 等。

MemberOf 模块功能可以指定 `groupOfUniqueNames` 用户组中通过 `uniquemember` 属性新增一个用户，LDAP 就会自动在该用户上创建一个 `memberOf` 属性，其值为该组的 dn

之前创建 ou(rd、op、pm) 的 objectClass 并没设置成 Group，LDAP 的 Group 是一种单独的类型 `objectClass: groupOfNames`，字段名为 `member`，value 就是 entry 的 dn，如此实现了 group & user 的映射关系。

## 安装模块

1）不同的发行版本，模块的路径可能不同，CentOS(yum)的模块路径为 `/usr/lib64/openldap`

```bash
$ find / -name memberof.la
/usr/lib64/openldap/memberof.la

$ ll /etc/openldap/slapd.d/cn\=config
total 24
drwxr-x--- 2 ldap ldap 4096 Jul 21 15:00 cn=schema
-rw------- 1 ldap ldap  378 Jul 21 14:59 cn=schema.ldif
-rw------- 1 ldap ldap  513 Jul 21 14:59 olcDatabase={0}config.ldif
-rw------- 1 ldap ldap  443 Jul 21 14:59 olcDatabase={-1}frontend.ldif
-rw------- 1 ldap ldap  606 Jul 21 15:01 olcDatabase={1}monitor.ldif
-rw------- 1 ldap ldap  966 Jul 21 15:01 olcDatabase={2}hdb.ldif
```

2）创建 `cn=module{0}` 文件，加载 memberOf 模块

> LDAP 安装后并没有 `Module` 的数据，需要单独建立

```bash
$ cat memberof.ldif
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
注：老的版本是 `groupOfNames,member`，新版本是 `groupOfUniqueNames,uniqueMember`

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

3）添加 refint，这个是 memberOf 的第一个模块

```bash
$ cat refint1.ldif
dn: cn=module{0},cn=config
add: olcmoduleload
olcmoduleload: refint

$ cat refint2.ldif
dn: olcOverlay=refint,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: refint
olcRefintAttribute: memberof uniqueMember manager owner
```

```bash
$ ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f refint1.ldif
$ ldapadd -Q -Y EXTERNAL -H ldapi:/// -f refint2.ldif

$ ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config dn | grep olcOverlay
dn: olcOverlay={0}memberof,olcDatabase={2}hdb,cn=config
dn: olcOverlay={1}refint,olcDatabase={2}hdb,cn=config
```

## 验证模块

1）创建一个 `GitLab` 功能组，并将用户添加至此组

```bash
$ cat group_memberof.ldif
dn: cn=gitlab,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfUniqueNames
cn: gitlab
uniqueMember: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
uniqueMember: uid=op001,ou=op,ou=People,dc=yuikuen,dc=top
```

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f group_memberof.ldif 
adding new entry "cn=gitlab,ou=Group,dc=yuikuen,dc=top"
```

2）查验模块配置是否成功，用户是否已成功添加至此组

```bash
$ ldapsearch -LL -Y EXTERNAL -H ldapi:///  "(cn=gitlab)" -b dc=yuikuen,dc=top
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
version: 1

dn: cn=gitlab,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfUniqueNames
cn: gitlab
uniqueMember: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
uniqueMember: uid=op001,ou=op,ou=People,dc=yuikuen,dc=top
```

```bash
$ ldapsearch -LL -Y EXTERNAL -H ldapi:///  "(uid=rd001)" -b dc=yuikuen,dc=top memberof
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
version: 1

dn: uid=rd001,ou=rd,ou=People,dc=yuikuen,dc=top
memberOf: cn=gitlab,ou=Group,dc=yuikuen,dc=top
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220729135924.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220729140403.png)

