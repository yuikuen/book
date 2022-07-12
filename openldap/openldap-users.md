# OpenLDAP创建用户&用户组

> OpenLDAP 创建用户&组的方式除了之前安装教程的 ldif 文件自定义字段外，还有其他的快捷创建方式

- 方式一：创建新用户 ldif 文件并执行创建

```bash
$ slappasswd -h {sha} -s 123456
{SHA}fEqNCco3Yq9h5ZUglD3CZJT4lBs=

$ vim new_user.ldif
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

$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f /etc/openldap/myldif/new_user.ldif
```

- 方式二：通过创建系统用户，使用 migrate_passwd.pl 文件生成要添加用户和用户组的 ldif

```bash
# 创建系统组&组用户
$ groupadd ldapgroup1
$ useradd -g ldapgroup1 ldapuser1

# grep用户信息过滤并提取至指定文件
$ grep "ldapuser" /etc/passwd > /root/users
$ grep "ldapgroup" /etc/group > /root/groups

$ /usr/share/migrationtools/migrate_passwd.pl /root/users > /root/users.ldif
$ /usr/share/migrationtools/migrate_group.pl /root/groups > /root/groups.ldif

$ ldapadd -x -w admin@123 -D cn=Manager,dc=yuikuen,dc=top -f /root/users.ldif
$ ldapadd -x -w admin@123 -D cn=Manager,dc=yuikuen,dc=top -f /root/groups.ldif
```

- 方式三：通过 phpLDAPadmin 直接创建账号及相关属性
【注：因我之前已提前创建好，一般默认是只有管理员 cn=admin】
**先创建 OU 组织单元**

1)创建 OU 组：Groups 和 People，点击 `dc=yuikuen,dc=top`，然后点击创建一个子条目

![image-20220629112645781](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629112645781.png)

2)点击 `Generic Organisational Unit`

![image-20220629112928811](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629112928811.png)

1. 输入 OU 名称为 Groups，然后创建对象，直接点击提交，完成提交之后 Groups 就创建成功

![image-20220629113156909](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629113156909.png)

![image-20220629113305800](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629113305800.png)

![image-20220629113349183](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629113349183.png)

4. Users 组按照上述步骤操作或者直接点击 ou=Groups，点击`复制和移该条目`，然后修改目标 DN 为 `ou=Users,dc=yuikuen,dc=top`，然后点击复制，提交即可；

![image-20220629113621195](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629113621195.png)

![image-20220629113701137](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629113701137.png)

![image-20220629113727636](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629113727636.png)

**创建 User 用户：rd001，op001，pm001，qa001（区分各权限用户）**

1. 点击 ou=Users，然后点击创建一个子条目

![image-20220629120752825](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629120752825.png)

2. 点击默认，选择 `inetOrgPerson`，然后点击继续

![image-20220629121230640](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629121230640.png)

3. 填写信息，其中 RDN 要选对，cn 和 sn 必须填写，其他的按实际情况填写，完成后直接点击创建

![image-20220629121445853](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629121445853.png)
![image-20220629121549664](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629121549664.png)
![image-20220629121655386](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629121655386.png)
![image-20220629121732058](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629121732058.png)
![image-20220629121753144](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629121753144.png)

4. 提交之后则已经创建成功，查看创建好的 rd001，可在此页面进行更新用户信息

![image-20220629122026186](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220629122026186.png)

5. 创建 op001，选择 cn=rd001，点击复制和移动该条目

![image-20220630172637093](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630172637093.png)

6. 修改目标 DN 的信息，然后点击复制

![image-20220630173435888](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630173435888.png)

7. 修改新用户 op001 的其他信息，然后点击创建对象

![image-20220630173555816](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630173555816.png)

8. 创建后可以查看用户列表，已经有两个用户

![image-20220630173649666](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630173649666.png)

**Groups 下创建用户组：JenkinsUsers 和 SonarUsers**

1. 点击 ou=Groups，点击创建一个子条目

![image-20220630173949385](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630173949385.png)

2. 点击默认，选择 groupOfNames，然后点击继续

![image-20220630174059098](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630174059098.png)

3. RDN 选择 cn(cn)，填写 cn 和 member，如下图所示，然后点击创建对象
   注意：menber 可任意填写，创建成功后可修改，箭头为绿色的代表填写正确

![image-20220630181037497](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630181037497.png)
![image-20220630181057255](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630181057255.png)

4. 创建 SonarUsers，选择 cn=JenkinsUsers，点击复制和移动该条目

![image-20220630181449194](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630181449194.png)

5. 修改目标 DN 的信息如下，然后点击复制

![image-20220630181721428](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630181721428.png)

![image-20220630181811815](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630181811815.png)

6. 最后就可以看到成功创建的组和用户

![image-20220630181957102](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630181957102.png)

**用户绑定指定用户组**

1. 点击 cn=SonarUsers，然后点击 modify group members

![image-20220630182222325](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630182222325.png)

2. 选择 rd001 条目，然后点击加入选择，最后点击 update Object 即可加入此组

![image-20220630182350962](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630182350962.png)
![image-20220630182409229](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630182409229.png)
![image-20220630182426076](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630182426076.png)

3. 回到页面，查看 SonarUsers 组，可以看到 rd001 已是组内成员

![image-20220630182451050](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220630182451050.png)