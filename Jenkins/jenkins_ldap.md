# Jenkins LDAP

> Jenkins 集成 OpenLDAP 账户，实现统一认证管理

## LDAP 配置

1）提前创建功能组并绑定用户

```bash
$ cat user_jenkins.ldif
dn: uid=pm001,ou=pm,ou=People,dc=yuikuen,dc=top
uid: pm001
cn: pm001
sn: pm001
givenName: xiong
displayName: lihua.xiong
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
uidNumber: 3000
gidNumber: 3000
gecos: System Manager
loginShell: /bin/bash
homeDirectory: /home/ldapusers
userPassword: {SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=
shadowLastChange: 17654
shadowMin: 0
shadowMax: 999999
shadowWarning: 7
shadowExpire: -1
employeeNumber: 30001
homePhone: 0769-xxxxxxxx
mobile: 181xxxxxxxx
mail: pm001@yuikuen.top
postalAddress: DongGuan
initials: Pm_Engineer

dn: cn=Jenkins,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfUniqueNames
cn: Jenkins
uniqueMember: uid=pm001,ou=pm,ou=People,dc=yuikuen,dc=top
uniqueMember: uid=op001,ou=op,ou=People,dc=yuikuen,dc=top
```

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f user_jenkins.ldif 
adding new entry "uid=pm001,ou=pm,ou=People,dc=yuikuen,dc=top"

adding new entry "cn=Jenkins,ou=Group,dc=yuikuen,dc=top"
```

## Jenkins 配置

> 注意：版本必须 \geq 2.277.x，并且要 \leq 2.289.x

1）下载 `ldap` 插件

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220802162053.png)

2）Configure Global Security(全局安全配置) --> Security Realm 填写 LDAP 服务信息进行关联

- LDAP 服务器地址：`ldap://ldap.yuikuen.top:389`
- 管理域 adminDN 账号：`cn=Manager,dc=yuikuen,dc=top`
- 基础域，用户查询用：`dc=yuikuen,dc=top`

