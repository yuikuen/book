# 服务部署

## 前置环境
1. 环境配置，关闭 SELinux & Firewalld
```bash
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service
```

## 安装程序
1. 执行`Yum`命令进行下载安装
```bash
$ yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel migrationtools

# 查看现版本
$ slapd -VV
@(#) $OpenLDAP: slapd 2.4.44 (Aug 31 2021 14:48:49) $
        mockbuild@x86-02.bsys.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd
```
