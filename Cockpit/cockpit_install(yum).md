# Cockpit Install

> Cockpit 是一款基于 Web 的可视化管理工具，对一些常见的命令行管理操作都有界面支持，比如用户管理、防火墙管理、服务器资源监控等

测试环境为 CentOS 7 版本，此外 Cockpit 还支持其他 Linux 发行版，请自行体验；另程序可根据需要进行扩展，实现各应用监控

[官网](https://cockpit-project.org/running.html)
[扩展](https://cockpit-project.org/applications.html)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220222102955180.png)

## 程序安装

1）Cockpit 安装(具体功能介绍请参考官网)

```bash
$ yum install -y cockpit cockpit-storaged cockpit-networkmanager cockpit-packagekit cockpit-ostree cockpit-machines cockpit-podman cockpit-selinux cockpit-kdump cockpit-sosreport cockpit-docker cockpit-dashboard
```

可自行安装其他扩展来实现功能

```bash
$ yum install -y cockpit-composer cockpit-certificates cockpit-389-ds cockpit-session-recording cockpit-subscriptions cockpit-ovirt-dashboard cockpit-zfs
```

2）启动程序并开放服务

```bash
$ systemctl enable --now cockpit.socket
$ firewall-cmd --permanent --zone=public --add-service=cockpit
$ firewall-cmd --reload
```

3）浏览器访问 9090 端口测试

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220222104321113.png)

用户名和密码就是 linux 服务器的用户名和密码，登陆即可进入首页

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220222105217234.png)

## 功能介绍

- 仪表盘

其他服务器也安装 Cockpit，可将机器添加到监控列表

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220222105350227.png)

- 网络监控

可配置防火墙，开放或禁止服务、端口等功能

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220222105559611.png)

- 虚拟容器

运行镜像以及拉取新镜像

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220222105729292.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220222105856251.png)

- 还有更多的功能，在此就不再逐一介绍，请自行搭建体验。