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
uidNumber: 2000
gidNumber: 2000
gecos: System Manager
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

1）下载 `ldap` 插件

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220802162053.png)

2）Configure Global Security(全局安全配置) --> Security Realm 填写 LDAP 服务信息进行关联

> 配置可参考 [Jenkins 官网](https://plugins.jenkins.io/ldap/)

- **Server**：LDAP 服务器地址，IP/Hostname (注意协议端口)
- **root DN**：子节点的DN，表示从 LDAP 的根节点开始搜索(指搜索的根，非 LDAP 服务器的 Root DN)
- **User search base**：用户搜索库-缩小搜索范围，指定搜索的用户范围，可为 `ou=People` 的相对值
- **User search filter**：用户搜索过滤器-定义登陆的用户名对应 LDAP 的哪字段，`{0}`会自动替换为提交的用户名
- **Group search base**：群组搜索库-查找用户的组列表，查询以此字段包含组的组织单元，可为 `ou=Group`
- **Group search filter**：组搜索过滤器-具有指定名称的组，默认值为 `(& (cn={0}) (| (objectclass=groupOfNames) (objectclass=groupOfUniqueNames) (objectclass=posixGroup)))`
- **Group membership**：组成员-确定用户所属的 LDAP 组，搜索包含用户的组(默认)，解析组列表的用户属性
- **Manager DN**：LDAP DN 管理员账户
- **Manager Password**：LDAP DN 管理员密码
- **Display Name LDAP attribute**：用户的显示名称，可自定义
- **Email Address LDAP attribute**：用户的邮箱对应的字段属性

以下图例所示，最终的效果是一样的，具体根据自身需求来配置

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220804140654.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220804140734.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220804140838.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220804140903.png)

使用 LDAP 账户登录后，可以在 Jenkins 用户列表进行查看

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220804141147.png)

