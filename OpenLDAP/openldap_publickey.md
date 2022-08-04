# OpenLDAP PublicKey

> 实际生产环境并不推荐用户直接明文密码登录，建议使用密钥登录，下面将演示 LDAP 账户通过密钥登录

## 服务端

1）LDAP 服务端安装 `openssh-ldap` 组件

> OpenLDAP 默认是没 ldapPublicKey，所以账户无法基于 sshkey 认证登录

```bash
$ yum -y install openssh-ldap
$ rpm -aql | grep openssh-ldap
/usr/share/doc/openssh-ldap-7.4p1
/usr/share/doc/openssh-ldap-7.4p1/HOWTO.ldap-keys
/usr/share/doc/openssh-ldap-7.4p1/ldap.conf
/usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-openldap.ldif
/usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-openldap.schema
/usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-sun.ldif
/usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-sun.schema
```

2）将配置复制至 Schema 目录，并添加组件

```bash
$ cp /usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-openldap.ldif /etc/openldap/schema/
$ cp /usr/share/doc/openssh-ldap-7.4p1/openssh-lpk-openldap.schema /etc/openldap/schema/

$ ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openssh-lpk-openldap.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=openssh-lpk,cn=schema,cn=config"
```

## 客户端

> 注：此处的客户端指的是其他应用服务器，非个人客户端工作站

1）同样安装 `openssh-ldap` 组件

```bash
$ yum -y install openssh-ldap
```

2）修改 `ldap.conf` 和 `sshd_config` 文件，支持 PubKey 登录

```bash
$ cp /usr/share/doc/openssh-ldap-7.4p1/ldap.conf /etc/ssh/
$ vim /etc/ssh/ldap.conf
# 未使用TLS,此处改为no
ssl no
uri ldap://188.188.4.111/

$ vim /etc/ssh/sshd_config +43
# 取消注释并按下述修改，此处为通过ssh-ldap-wrapper脚本获取密钥并将其提供给SSH服务验证
 43 PubkeyAuthentication yes
 44 
 45 # The default is to check both .ssh/authorized_keys and .ssh/authorized_keys2
 46 # but this is overridden so installations will only check .ssh/authorized_keys
 47 AuthorizedKeysFile      .ssh/authorized_keys
 48 
 49 #AuthorizedPrincipalsFile none
 50 
 51 AuthorizedKeysCommand /usr/libexec/openssh/ssh-ldap-wrapper
 52 AuthorizedKeysCommandUser nobody
```

## 登录验证

> 服务器生成密钥文件，并将公钥绑定到指定用户的 `sshPublicKey` 属性值即可

**操作说明：**

网上大多数的教程都未特别注明密钥的来源，或只是写着“我们需要登录的服务器”，因其工作原理跟日常的服务器生成密钥，用户再使用该私钥文件进行 SSH-Key 认证登录一样，只是多了绑定 LDAP 账户的操作。

虽然是实现了 LDAP 统一 SSH-Key 认证登录，但在实际生产环境中，对于运维管理并不便利，下面将演示我是如何实现统一 SSH-Key 认证管理。

1）LDAP 服务端生成密钥

```bash
$ mkdir ~/.ssh
$ ssh-keygen -t rsa -f ~/.ssh/ldap_server
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/ldap_server.
Your public key has been saved in /root/.ssh/ldap_server.pub.
The key fingerprint is:
SHA256:dLVSD2bEYiIILDCmJQb1zQmiAKPBG5v/lzIKDs5SItI root@ssd-dev01
The key's randomart image is:
+---[RSA 2048]----+
|#==...     oB    |
|BX.o.+... o=.+   |
|+.= . +..oo.. .  |
| +     . . .     |
| ..     S        |
|+ E.             |
|++  .   .        |
|* .  + o         |
|.+ .. +          |
+----[SHA256]-----+

$ ls -al ~/.ssh/
total 8
-rw-------  1 root root 1675 Aug  1 14:16 ldap_server
-rw-r--r--  1 root root  396 Aug  1 14:16 ldap_server.pub
```

2）LDAP 指定账户添加 `objectClass: ldapPublicKey` 并添加属性 `sshPublicKey`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220801141840.png)

选择 ldapPublicKey 后会提示输入 sshPublicKey 值，添加 LDAP 公钥就可以了

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220801141916.png)

3）客户端(Pc工作站)使用 LDAP 服务端的私钥 + Xshell 进行登录操作

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220801142643.png)

**注意：此处已成功使用 LDAP 服务器的私钥登录任意已关联 LDAP 服务的服务器**

## 实验总结

> 具体方案可根据自身需求来定

为什么要有实验总结？首先说明下，之前我也是按照常规操作，在每个服务器上进行生成密钥，之后绑定到"指定的用户"，实现了指定 LDAP 账户拥有 SSH-Key 访问"指定的服务器"。但这样不仅需要维护 LDAP 账户，还需要记录不同服务器的密钥文件，这对于运维同事来说，反而增加了工作量。

目前的管理方案是通过 LDAP 服务端生成密钥文件，将公钥绑定到指定 LDAP 账户，并且限制指定账户登录指定的服务器主机(此处的服务器必须按照上述客户端配置组件服务)。