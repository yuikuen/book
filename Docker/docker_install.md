# Docker Install(yum)

## 版本环境

- System：CentOS7.9.2009 Minimal
- Runtime：docker-ce-20.10.*

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装 Docker

> [Docker](https://docs.docker.com/engine/install/centos/) 要求 CentOS 系统的内核版本高于 3.10，安装前提前升级内核

1）安装需要的软件包，`yum-util` 提供 `yum-config-manager` 功能，另外两个是 `devicemapper` 驱动依赖

```bash
$ yum -y install yum-utils device-mapper-persistent-data lvm2
```

2）设置存储库

```bash
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

由于国内访问不到 Docker 官方镜像，可以通过 aliyun 的源来完成

```bash
$ yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
$ yum makecache fast
```

3）安装程序，默认最新版(**docker-ce:社区版 ee:企业版**)

```bash
$ yum list docker-ce --showduplicates | sort -r
 * updates: mirrors.aliyun.com
Loading mirror speeds from cached hostfile
Loaded plugins: fastestmirror
 * extras: mirrors.ustc.edu.cn
docker-ce.x86_64            3:20.10.9-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.8-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.7-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.6-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.5-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.4-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.3-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.2-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.17-3.el7                    docker-ce-stable
docker-ce.x86_64            3:20.10.16-3.el7                    docker-ce-stable
docker-ce.x86_64            3:20.10.15-3.el7                    docker-ce-stable
docker-ce.x86_64            3:20.10.14-3.el7                    docker-ce-stable
docker-ce.x86_64            3:20.10.1-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.13-3.el7                    docker-ce-stable
docker-ce.x86_64            3:20.10.12-3.el7                    docker-ce-stable
docker-ce.x86_64            3:20.10.11-3.el7                    docker-ce-stable
docker-ce.x86_64            3:20.10.10-3.el7                    docker-ce-stable
docker-ce.x86_64            3:20.10.0-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.9-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.8-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.7-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.6-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.5-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.4-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.3-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.2-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.15-3.el7                    docker-ce-stable
docker-ce.x86_64            3:19.03.14-3.el7                    docker-ce-stable
docker-ce.x86_64            3:19.03.1-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.13-3.el7                    docker-ce-stable
docker-ce.x86_64            3:19.03.12-3.el7                    docker-ce-stable
docker-ce.x86_64            3:19.03.11-3.el7                    docker-ce-stable
docker-ce.x86_64            3:19.03.10-3.el7                    docker-ce-stable
docker-ce.x86_64            3:19.03.0-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.9-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.8-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.7-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.6-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.5-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.4-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.3-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.2-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.1-3.el7                     docker-ce-stable
docker-ce.x86_64            3:18.09.0-3.el7                     docker-ce-stable
docker-ce.x86_64            18.06.3.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.2.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.0.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            18.03.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.09.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.09.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.2.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.06.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.3.ce-1.el7                    docker-ce-stable
docker-ce.x86_64            17.03.2.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.03.0.ce-1.el7.centos             docker-ce-stable
 * base: mirrors.aliyun.com
Available Packages
```

安装指定版本的 Docker-CE: (VERSION 例如上面的 3:20.10.17-3.el7)

```bash
$ yum -y install docker-ce-[VERSION]
$ yum -y install docker-ce-20.10.* docker-cli-20.10.*

$ docker version
Client: Docker Engine - Community
 Version:           20.10.17
 API version:       1.41
 Go version:        go1.17.11
 Git commit:        100c701
 Built:             Mon Jun  6 23:05:12 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

4）设置防火墙规则，让其通行(可略，根据实际环境操作)

```bash
$ vim /lib/systemd/system/docker.service
[Service]
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
```

5）启动服务

```bash
$ systemctl enable --now docker
```

## 配置加速

> 镜像加速器：阿里云加速器，daocloud 加速器，中科大加速器，Docker 中国官方镜像等

```bash
$ cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": [
      "https://eihzr0te.mirror.aliyuncs.com",
      "https://dockerhub.mirrors.nwafu.edu.cn/",
      "https://mirror.ccs.tencentyun.com",
      "https://docker.mirrors.ustc.edu.cn/",
      "https://reg-mirror.qiniu.com",
      "http://hub-mirror.c.163.com/",
      "https://registry.docker-cn.com"]
}
EOF

$ systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```

注：修改配置后需要重启相关服务
