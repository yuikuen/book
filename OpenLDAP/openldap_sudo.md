# OpenLDAP Sudo

> OpenLDAP 默认安装的 Schema 目录，是没有 sudo.ldif 以及 sudo.schema 文件，需要另外配置

## 服务端

1）查找组件文件并复制至 Schema 目录

```bash
$ find / -name schema.OpenLDAP
/usr/share/doc/sudo-1.8.23/schema.OpenLDAP

$ find / -name schema.OpenLDAP -exec cp {} /etc/openldap/schema/sudo.schema \;
$ ls -l /etc/openldap/schema | grep sudo.*
-rw-r--r--. 1 root root 2410 Aug  1 17:04 sudo.schema
```

2）再通过 `sudo.schema` 文件生成 `sudo.ldif` 文件

```bash
$ echo 'include /etc/openldap/schema/sudo.schema' > /tmp/sudo.conf
$ mkdir -p /tmp/sudo
$ slaptest -f /tmp/sudo.conf -F /tmp/sudo
```

3）修改已生成好的 `sudo.ldif` 配置文件

```bash
$ cd /tmp/sudo/cn=config/cn=schema
$ cat cn={0}sudo.ldif
# 替换前三行内容
dn: cn={0}sudo
objectClass: olcSchemaConfig
cn: {0}sudo
# 将上述的改为下面的内容
dn: cn=sudo,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: sudo

# 删除最后的七行内容
structuralObjectClass: olcSchemaConfig
entryUUID: ec3b659a-31a9-1039-90ae-87c69280e4a2
creatorsName: cn=config
createTimestamp: 20190703064542Z
entryCSN: 20190703064542.945991Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20190703064542Z
```

4）修改后进行加载配置

```bash
$ cp /tmp/sudo/cn=config/cn=schema/cn={0}sudo.ldif /etc/openldap/schema/sudo.ldif
$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/sudo.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=sudo,cn=schema,cn=config"
```

## 权限配置

1）创建一个权限管理的组织单元

```bash
$ touch ./myldif/sudoers.ldif; vim sudoers.ldif
dn: ou=sudoers,dc=yuikuen,dc=top
ou: sudoers
objectClass: top
objectClass: organizationalUnit

dn: cn=defaults,ou=sudoers,dc=yuikuen,dc=top
objectClass: sudoRole
cn: defaults
sudoOption: requiretty
sudoOption: !visiblepw
sudoOption: always_set_home
sudoOption: env_reset
sudoOption: env_keep =  "COLORS DISPLAY HOSTNAME HISTSIZE INPUTRC KDEDIR LS_COLORS"
sudoOption: env_keep += "MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE"
sudoOption: env_keep += "LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES"
sudoOption: env_keep += "LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE"
sudoOption: env_keep += "LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY"
sudoOption: secure_path = /sbin:/bin:/usr/sbin:/usr/bin
sudoOption: logfile = /var/log/sudo
```

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f sudoers.ldif
adding new entry "ou=sudoers,dc=yuikuen,dc=top"
adding new entry "cn=defaults,ou=sudoers,dc=yuikuen,dc=top"
```

2）创建 sudo 权限规则组(Role)，并将账户添加到该组

```bash
$ touch ./myldif/sudo_role.ldif; vim sudo_role.ldif
dn: cn=sudo_role,ou=sudoers,dc=yuikuen,dc=top
objectClass: sudoRole
cn: sudo_role
sudoOption: !authenticate
sudoRunAsUser: root
sudoCommand: ALL
sudoHost: ALL
sudoUser: op001
```

```bash
$ ldapadd -x -w "Admin@123" -D "cn=Manager,dc=yuikuen,dc=top" -f sudo_role.ldif
adding new entry "cn=sudo_role,ou=sudoers,dc=yuikuen,dc=top"
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220801173100.png)

## 客户端

1）客户端服务器添加 sudo 权限配置

```bash
$ cat /etc/nsswitch.conf
# 文末添加
sudoers:    files ldap

$ cat /etc/sudo-ldap.conf | grep -v '#' | grep -v '^$'
uri ldap://188.188.4.111/
sudoers_base ou=sudoers,dc=yuikuen,dc=top
```

2）验证测试，SSH 上述已修改权限配置的客户端服务器

```bash
Connecting to 188.188.4.112:22...
Connection established.
To escape to local shell, press 'Ctrl+Alt+]'.

WARNING! The remote SSH server rejected X11 forwarding request.
Last login: Mon Aug  1 14:23:27 2022 from ec2-100-20-0-217.us-west-2.compute.amazonaws.com
[Mon Aug 01 root@ssd-dev02 ~]# ssh op001@188.188.4.111
op001@188.188.4.111's password: 
Last login: Mon Aug  1 17:37:58 2022 from ec2-100-20-0-217.us-west-2.compute.amazonaws.com
/usr/bin/id: cannot find name for group ID 1000
[Mon Aug 01 op001@ssd-dev01 ~]$ sudo su -
Last login: Mon Aug  1 17:38:15 CST 2022 on pts/1
[Mon Aug 01 root@ssd-dev01 ~]# id
uid=0(root) gid=0(root) groups=0(root)
```

## 配置模板

> 默认安装的 PhpLDAPAdmin & LDAP 是没有创建模板的，需要自行添加

1）首先到 PhpLDAPAdmin 官网获取 [sudoRole模版](http://phpldapadmin.sourceforge.net/wiki/index.php/TemplatesContributed:Sudo)

```bash
$ cd /usr/share/phpldapadmin/templates
$ touch creation/sudo.xml 
$ touch modification/sudo.xml
```

根据实际需求进行修改参数字段等数值

```xml
$ cat creation/sudo.xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE template SYSTEM "template.dtd">
<template>
<title>Sudo Policy</title>
<regexp>^ou=sudoers,dc=.*</regexp>
<icon>images/door.png</icon>
<description>New Sudo Policy</description>
<askcontainer>1</askcontainer>
<rdn>cn</rdn>
<visible>1</visible>

<objectClasses>
<objectClass id="sudoRole"></objectClass>
</objectClasses>

<attributes>
<attribute id="cn">
        <display>Policy Name</display>
        <order>1</order>
        <page>1</page>
</attribute>
<attribute id="sudoCommand">
        <display>Sudo Command</display>
        <order>2</order>
        <page>1</page>
        <spacer>1</spacer>
</attribute>
<attribute id="sudoUser">
        <display>Sudo Users</display>
        <option>=php.MultiList(/,(objectClass=posixAccount),uid,%uid%
(%cn%),sudoUser)</option>
        <order>3</order>
        <page>1</page>
        <spacer>1</spacer>
</attribute>
<attribute id="sudoHost">
        <display>Sudo Hosts</display>
        <array>10</array>
        <order>3</order>
        <page>1</page>
        <spacer>1</spacer>
</attribute>
<attribute id="description">
        <type>textarea</type>
        <display>Description</display>
        <order>4</order>
        <page>1</page>
</attribute>
</attributes>
</template>
```

```xml
$ cat modification/sudo.xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE template SYSTEM "template.dtd">
<template>
<title>Sudo Policy</title>
<regexp>^cn=.*,ou=sudoers,dc=.*</regexp>
<icon>images/door.png</icon>
<description>Sudo Policy</description>
<askcontainer>1</askcontainer>
<rdn>cn</rdn>
<visible>1</visible>

<objectClasses>
<objectClass id="sudoRole"></objectClass>
</objectClasses>

<attributes>
<attribute id="cn">
        <display>Policy Name</display>
        <order>1</order>
        <page>1</page>
</attribute>
<attribute id="sudoCommand">
        <display>Sudo Command</display>
        <order>2</order>
        <page>1</page>
        <spacer>1</spacer>
</attribute>
<attribute id="sudoUser">
        <display>Sudo Users</display>
        <order>3</order>
        <page>1</page>
        <spacer>1</spacer>
</attribute>
<attribute id="sudoHost">
        <display>Sudo Hosts</display>
        <!-- <array>10</array> -->
        <order>3</order>
        <page>1</page>
        <spacer>1</spacer>
</attribute>
<attribute id="description">
        <type>textarea</type>
        <display>Description</display>
        <order>4</order>
        <page>1</page>
        <cols>200</cols>
        <rows>10</rows>
</attribute>
</attributes>
</template>
```

2）重启 httpd 服务，查看子目录模板是否有 `sudoRole`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220801175210.png)