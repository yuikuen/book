# Gitlab集成ldap登录

> 之前已安装过 OpenLDAP 了，现简单介绍下 Gitlab 与 OpenLDAP 集成，实现统一认证管理

## 集成登录

1)修改 `gitlab.rb` 配置文件，启用 ldap 登录验证方式

```bash
$ vim /etc/gitlab/gitlab.rb +475
# 修改服务为true,启动ldap登录方式
gitlab_rails['ldap_enabled'] = true

# 通过EOS把服务配置包起来
gitlab_rails['ldap_servers'] = YAML.load <<-'EOS'
  main: # 'main' is the GitLab 'provider ID' of this LDAP server
    label: 'LDAP140'                        # LDAP服务标签名，可以自行命名
    host: '188.188.4.140'                   # Server_ip or Server_hostname
    port: 389                               # 默认端口，ssl一般为636
    uid: 'cn'                               # 这里指的是登陆时用的是LDAP信息中的那个字段，一般来说，是类似于用户名、用户ID这样的字段，根据实际情况修改
    bind_dn: 'cn=Manager,dc=yuikuen,dc=top' # 绑定的LDAP完整的DN路径
    password: 'Admin@123'                   # 绑定的用户密码，这指的是Manager的密码
    encryption: 'plain'                     # 加密方式"start_tls" or "simple_tls" or "plain"
    verify_certificates: true               # 如果加密方式为SSL,这里的验证会生效
    smartcard_auth: false                   # 认证方式
    active_directory: true                  # 判断是不是Active Diretory类型的LDAP服务
    allow_username_or_email_login: true     # 登陆方式用户名或邮箱
    lowercase_usernames: false
    block_auto_created_users: false
    base: 'dc=yuikuen,dc=top'               # 以此为基础，进行用户查询
    user_filter: ''                         # 表示以某种过滤条件筛选用户
    # 下面的参数指定GitLab会使用LDAP中的哪些属性值作为自己的用户信息，比如GitLab中的username信息会
    # 按照先后顺序匹配LDAP中的'uid'/'userid'/'sAMAccountName'属性值中的一个，fist_name字段就直接
    # 对应LDAP中的'givenName'属性
    attributes:                             # 示意gilab的字段与ldap中哪些字段能够互相对应
      username: ['cn']
      email: ['mail']
      name: 'displayName'
      first_name: 'givenName'
      last_name: 'sn'
    ## EE only
    group_base: ''
    admin_group: ''
    sync_ssh_keys: false
EOS
```
注：其中 Secondary 的配置是 OpenLDAP 主从模式时使用，现暂不详细解析

![image-20220704115609550](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704115609550.png)

2)配置完毕后必须重新加载配置，再执行检查命令确认配置是否成功

```bash
$ gitlab-ctl reconfigure
$ gitlab-rake gitlab:ldap:check
Checking LDAP ...

LDAP: ... Server: ldapmain
LDAP authentication... Success
LDAP users with access to your GitLab server (only showing the first 100 results)
	DN: cn=manager,dc=yuikuen,dc=top	 cn: Manager
	DN: cn=rd001,ou=rd,ou=people,dc=yuikuen,dc=top	 cn: rd001
	DN: cn=op001,ou=op,ou=people,dc=yuikuen,dc=top	 cn: op001
	DN: cn=pm001,ou=pm,ou=people,dc=yuikuen,dc=top	 cn: pm001
	DN: cn=qa001,ou=qa,ou=people,dc=yuikuen,dc=top	 cn: qa001
	DN: cn=rd,ou=group,dc=yuikuen,dc=top	 cn: rd
	DN: cn=op,ou=group,dc=yuikuen,dc=top	 cn: op
	DN: cn=pm,ou=group,dc=yuikuen,dc=top	 cn: pm
	DN: cn=qa,ou=group,dc=yuikuen,dc=top	 cn: qa

Checking LDAP ... Finished
```
**注意：出现 Success 并不代表已成功配置，必须正常输出用户列表信息，才算配置成功！！**

3)打开浏览器，使用上述罗列出来的用户进行登录

![image-20220704120109359](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704120109359.png)

![image-20220704120142298](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704120142298.png)

## 过滤登录

> 通过上述演示，可以看到 Gitlab 集成 OpenLDAP 已成功。在 check 命令下，可以看到 ldap 所有用户，这也表明所有用户都能登录 Gitlab。
> 在实际的生产环境，代码仓库应由专门的人员进行维护，所以上述方法并不适用于生产上。其实也可以使用 Gitlab 原生的管理员进行用户的管理，但需专人去维护用户权限，这并不高效。下面将演示如何限制于指定用户组用户方可访问代码仓库。

![image-20220704120539654](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704120539654.png)

1)首先回到 OpenLDAP 服务器，添加一个专用组（可复制之前的 addgroup.ldif 进行修改）

```bash
$ vim gitlabUsers.ldif
# 创建一个为gitlabUsers的组，并将ou=op下的系统管理员op001绑定至此组
dn: cn=gitlabUsers,ou=Group,dc=yuikuen,dc=top
objectClass: groupOfNames
cn: gitlabUsers
member: cn=op001,ou=op,ou=People,dc=yuikuen,dc=top
```

2)执行创建命令，将 `cn=op001` 绑定至 `cn=gitlabUsers`

```bash
$ ldapmodify -a -H ldap://188.188.4.140:389 -D "cn=Manager,dc=yuikuen,dc=top" -w Admin@123 -f gitlabUsers.ldif
adding new entry "cn=gitlabUsers,ou=Group,dc=yuikuen,dc=top"
```

![image-20220704122121237](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704122121237.png)

3)修改 `gitlab.rb` 配置文件中的 `user_filter` 参数，仅允许 gitlabUsers 组的用户才能登录

```bash
# 参照上述进行服务配置
gitlab_rails['ldap_enabled'] = true

# 通过EOS把服务配置包起来
gitlab_rails['ldap_servers'] = YAML.load <<-'EOS'
  main: # 'main' is the GitLab 'provider ID' of this LDAP server
    label: 'LDAP140'
    host: '188.188.4.140'
    port: 389
    uid: 'cn'
    bind_dn: 'cn=Manager,dc=yuikuen,dc=top'
    password: 'Admin@123'
    encryption: 'plain'
    verify_certificates: true
    smartcard_auth: false
    active_directory: true
    allow_username_or_email_login: true
    lowercase_usernames: false
    block_auto_created_users: false
    base: 'dc=yuikuen,dc=top'
    # 其他都不变，只是在此增加memberOf值
    user_filter: '(memberOf=cn=gitlabUsers,ou=Group,dc=yuikuen,dc=top)'
    attributes:
      username: ['cn']
      email: ['mail']
      name: 'displayName'
      first_name: 'givenName'
      last_name: 'sn'
    ## EE only
    group_base: ''
    admin_group: ''
    sync_ssh_keys: false
EOS
```
注：此处的过滤功能，就是之前安装 OpenLDAP 的 memberof 模块功能，实现分组认证

4)再次加载配置，并刷新检查，此时会发现输出的用户列表只有 `cn=op001`

```bash
$ gitlab-ctl reconfigure
$ gitlab-rake gitlab:ldap:check
Checking LDAP ...

LDAP: ... Server: ldapmain
LDAP authentication... Success
LDAP users with access to your GitLab server (only showing the first 100 results)
	DN: cn=op001,ou=op,ou=people,dc=yuikuen,dc=top	 cn: op001

Checking LDAP ... Finished
```

5)再次使用 ldap 账号登录，此时非 `cn=gitlabUsers` 的用户都无法登录 Gitlab

![image-20220704155411438](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704155411438.png)

*小知识：*
除了自建 ldif 文件的方式，也可在 PhpLDAPAdmin 界面下直接新增、复制、添加等操作

![image-20220704162135689](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704162135689.png)

![image-20220704162215353](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704162215353.png)

![image-20220704162303418](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220704162303418.png)