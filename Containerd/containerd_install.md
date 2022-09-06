# Containerd Install(SRC)

> CentOS7 二进制部署 Containerd 1.6.x

## 版本环境

- System：CentOS7.9.2009 Minimal
- Containerd：cri-containerd-cni-1.6.8-linux-amd64

## 安装 libseccomp

> CentOS7 默认 libseccomp 版本是 2.3.x，版本不满足最新版的要求

1）卸载旧版本程序

```bash
$ rpm -qa | grep libseccomp

$ rpm -e libseccomp-* --nodeps
```

2）下载高于 2.4 以上的包

```bash
$ wget http://rpmfind.net/linux/centos/8-stream/BaseOS/x86_64/os/Packages/libseccomp-2.5.1-1.el8.x86_64.rpm
$ rpm -ivh libseccomp-2.5.1-1.el8.x86_64.rpm 
$ rpm -qa | grep libseccomp
```

## 安装 Containerd

> `cri-containerd-1.x.x-linux-amd64.tar.gz` 包含 containerd 以及 cri runc 等相关工具包，建议下载

1）根据需要到 [GitHub 官方地址](https://github.com/containerd/containerd) 下载 `tar.gz` 包

```bash
$ wget https://github.com/containerd/containerd/releases/download/v1.6.8/cri-containerd-1.6.8-linux-amd64.tar.gz
```

2）解压并修改环境变量

```bash
$ tar -zxvf cri-containerd-1.6.8-linux-amd64.tar.gz -C /

$ export PATH=$PATH:/usr/local/bin:/usr/local/sbin
$ source ~/.bashrc
```

3）创建 Containerd 配置文件

```bash
$ mkdir -p /etc/containerd 
$ containerd config default > /etc/containerd/config.toml
```

由于 containerd 压缩包中已包含了 `etc/systemd/system/containerd.service` 文件，可直接启动服务

```bash
$ cat /etc/systemd/system/containerd.service

$ systemctl enable --now containerd
$ ctr version
```
