# Nexus3 Roles

> 创建权限策略，分配指定用户权限，策略规则请参考 [官网文档](https://help.sonatype.com/repomanager3/nexus-repository-administration/access-control/roles)

Nexus3 已与 LDAP 集成，实现统一用户管理，但登录后发现未有任何功能。此时还需要为用户创建并分配指定权限策略方可使用

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826145330.png)

1）使用管理员登录，选中 `configuration`-->`Security`-->`Roles`，点击 `Create Role` 创建策略

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826145921.png)

2）选择 `Role Type` 角色类型

> 创建角色的方法其实是一样，主要是 ID、Name、Privileges、Roles

- Nexus role：内部角色
- External Role Mapping：外部组映射角色

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826151005.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826151033.png)

3）用户分配权限，选择 `Users`-->`LDAP`

默认显示的是内置角色的用户，需要切换至对应的权限组

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826151114.png)

选中需要配置的用户，并添加指定的角色权限策略

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826151413.png)

退出管理员，再次使用 `op001` 进行登录，可以查看对应的功能

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826151544.png)