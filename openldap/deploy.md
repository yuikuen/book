# 服务部署

## 前置环境
1. 环境配置，关闭 SELinux & Firewalld
```bash
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service
```

## 安装程序
1. 执行 `Yum` 命令进行下载安装
```bash
$ yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel migrationtools

# 查看现版本
$ slapd -VV
@(#) $OpenLDAP: slapd 2.4.44 (Aug 31 2021 14:48:49) $
        mockbuild@x86-02.bsys.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd
```

2. 生成管理员密码
```bash
$ slappasswd -s Admin@123
{SSHA}GJEm8DDGliPDM+/ip/lk5d+KDBGjR9TU
```

3. 修改配置文件，添加刚生成的密码
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

4. 验证配置文件是否正确
```bash
$ slaptest -u
62c25987 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif"
62c25987 ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif"
config file testing succeeded
```
注：如出现 `checksum error on`，可忽略错误继续操作

5. 启动服务并设置开机自启
```bash
$ systemctl enable --now slapd && systemctl status slapd
$ netstat -anpl|grep 389
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      5488/slapd          
tcp6       0      0 :::389                  :::*                    LISTEN      5488/slapd
```

## 配置程序
> 安装 openldap 后，会有三个命令用于修改配置文件，分别为 ldapadd，ldapmodify，ldapdelete，顾名思义就是添加，修改和删除。需要修改或增加配置时，则需要先写一个ldif后缀的配置文件，然后通过命令将写的配置更新到 slapd.d 目录下的配置文件中去

1. 配置数据库，OpenLDAP 默认使用的数据库是 BerkeleyDB
```bash
$ cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
$ chown ldap:ldap -R /var/lib/ldap
$ chmod 700 -R /var/lib/ldap
```
注：`/var/lib/ldap` 为 BerkeleyDB 数据库默认存储的路径
