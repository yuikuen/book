# Esxi Shell(SSH)

> **远程映射图形管理，更加方便操作**
> 
> Esxi 管理一般都通过 Web 进行，但需要更改管理接口 IP 地址或其他操作，就需要登录管理后台

1）启动 Esxi 的安全 SHELL

Web 管理界面 --> 主机 --> 操作 --> 服务菜单，启用 `安全 SHELL(SSH)`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220820153604.png)

2）使用 SSH 工具登录

Win-OS 下使用 Xshell、PuTTY、MobaXterm 等工具，远程 SSH 管理后台，命令行下输入

```bash
$ dcui
```

Mac-OS 下使用自带终端工具进入，再命令行下输入如下

```bash
$ TERM=xterm 
$ dcui
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220820161319.png)

其他操作功能与实际显示终端操作一致。