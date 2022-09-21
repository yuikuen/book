# Docker Open-2375

> Docker 开启 2375 远程管理接口

## 概要说明

为了实现集群管理，Docker 提供了远程管理接口。Docker Daemon 作为守护进程，运行在后台，可以执行发送到管理接口上的 Docker 命令。

而启动 Docker Daemon 时，加入 -H 0.0.0.0:2375，Docker Daemon 就可以接收远端的 Docker Client 发送的指令。

注意，Docker 是把 2375 端口作为非加密端口暴露出来，一般是用在测试环境中。此时，没有任何加密和认证过程，只要知道 Docker 主机的 IP，任何人都可以管理这台主机上的容器和镜像

## 修改配置

**Docker 开启 2375 端口，提供外部访问 Docker**

1）在 `ExecStart=/usr/bin/dockerd-current` 后增加 `-H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock`

```bash
$ vim /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock 
```

在此附上 `docker.service` 文件

```sh
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock -H fd:// --containerd=/run/containerd/containerd.sock
# 在此增加 tcp://0.0.0.0:2375
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
# 在此增加防火墙通行规则
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
```

## 验证服务

1）重启 docker，重新读取配置文件并启动服务

```bash
$ systemctl daemon-reload
$ systemctl restart docker
```

2）查看相关服务、状态是否正常

```bash
$ netstat -tunlp | grep 2375
tcp6       0      0 :::2375                 :::*                    LISTEN      5388/dockerd

# 如失败请检查防火墙及Selinux服务
$ firewall-cmd --add-port=2375/tcp
$ firewall-cmd --add-port=2375/tcp --permanent
```

注：上述说明已基本讲解了 Docker 开启 2375 端口的作用，一般应用在内网测试环境，而云上服务建议勿开启并暴露此端口，以防被作矿机