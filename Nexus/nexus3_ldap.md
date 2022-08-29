# Nexus3 LDAP

> 配置请参考 [官方文档](https://help.sonatype.com/repomanager3/nexus-repository-administration/user-authentication/ldap)
> 
> 提醒：本文供介绍 Nexus3 集成 OpenLDAP 账户，实现统一认证管理，但用户并无相应权限，需另设置使用权限

1）登录后选择 `server configuration`，在左侧菜单栏选中 `Security > LDAP`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826115208.png)

2）点击创建 `LDAP`，按照提示内容进行设置 `LDAP Connection`

> 首先配置 LDAP 服务器信息，配置后可点 `Verify connection` 测试连接

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826115434.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826115615.png)

**示例配置，仅供参考：**

|配置|参数|说明|
|--|--|--|
|Name|LDAP|自定义命名|
|LDAP server address|ldap://188.188.4.204|LDAP 服务器地址，可选 ldap/ldaps|
|Port|389/636|访问端口，根据上述协议设置|
|Search base DN|dc=yuikuen,dc=top|LDAP 基础域|
|Authentication method|Simple Authentication|身份验证方式|
|Username or DN|cn=Manager,dc=yuikuen,dc=top|具有管理权限的用户|
|Password|*|管理用户的密码|

3）下一步设置 `Choose Users and Groups` 用户 & 组信息

- 用户信息

> `User filter` 可为空，表示所有用户

|配置|参数|说明|
|--|--|--|
|Configuration template|Generic Ldap Server|用户和组的配置模板|
|User relative DN|ou=People|用户搜索基础 DN|
|User subtree|勾选|是否存在子树|
|Object class|inetOrgPerson|用户对象属性|
|User filter|ou=op*|用户过滤条件|
|User ID attribute|uid|用户提供标识符|
|Real name attribute|cn|用户名名称属性|
|Email attribute|mail|对象属性|
|Password attribute|置空|为空则用户通过 LDAP 绑定身份验证|

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826114536.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826114659.png)

注：不设置用户组，单配置用户信息也是可以使用的

- 用户组信息

> `Map LDAP groups as roles` 有分动态和静态组之分

- 动态组

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826141214.png)

- 静态组

|配置|参数|说明|
|--|--|--|
|Group relative DN|ou=Group|用户组搜索基础 DN|
|Group subtree|勾选|是否存在子树|
|Group object class|groupOfUniqueNames|组对象的LDAP类|
|Group ID attribute|cn|指定指定组标识符的对象类的属性|
|Group member attribute|uniqueMember|组成员属性指定对象类的属性|
|Group member format|${dn}|用户 ID 的格式|

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826143734.png)

注：未测试成功按组过滤，只实现了通过 `User filter` 配置 `(|(uid=rd*)(uid=op*))` 仅允许运维和开发使用


















