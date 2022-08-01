# OpenLDAP WebUi

> 通过上节 [OpenLDAP Install](01-openldap_install.md) 的安装配置，LDAP 服务已经可以使用了。
> 
> 为了更好的演示和管理，可以使用 `PhpLDAPAdmin` 或 `LDAP Admin` 等图形管理工具

## PhpLDAPAdmin

> 基于 Web 的 OpenLDAP 管理工具，通过代理实现

1）安装依赖包、服务组件，并下载程序

```bash
$ yum -y install httpd php php-ldap php-gd php-mbstring php-pear php-bcmath php-xml
$ yum -y install epel-release 
$ yum --enablerepo=epel -y install phpldapadmin
```

2）修改配置文件，开放访问权限

```bash
$ cat /etc/httpd/conf.d/phpldapadmin.conf
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

3）修改登录方式、关闭匿名登录及用户属性唯一等

> 注意：网上大多教程搭建后无法登录，因默认为uid登录方式
> 
> 请根据自身需求进行修改：dn-域名全称，cn-仅为用户名

```bash
$ vim /etc/phpldapadmin/config.php +398
# nu:398，修改默认登录方式
$servers->setValue('login','attr','dn');

# nu:460，将true改为false，关闭匿名登录
$servers->setValue('login','anon_bind',false);

# nu:520，设置用户属性的唯一，把cn,sn加上
$servers->setValue('unique','attrs',array('mail','uid','uidNumber','cn','sn'))
```

4）修改后重启服务，打开浏览器登录验证

> 以下是通过域名解析方式进行访问，所以需要保证域名功能正常使用，否则需要自行配置 `hosts` 文件
> 
> 此处为本地测试环境，自建 `Linux DNS` 服务

```bash
$ systemctl enable --now httpd && systemctl status httpd
```

- 地址：http://ldap.yuikuen.top/phpldapadmin
- 账号：cn=Manager,dc=yuikuen,dc=top
- 密码：Admin@123

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220728165733.png)

## LDAP Admin

> Windows 下的 OpenLDAP 管理工具，直接解压打开 exe 文件即可使用

1）到 [LDAP Admin官网](http://www.ldapadmin.org/index.html)，找到 [Download](https://sourceforge.net/projects/ldapadmin/files/ldapadmin/1.8.3/) 下载（当前最新版本是 1.8.3）

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220721161548513.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220721161705076.png)