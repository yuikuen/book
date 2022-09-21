

## 基础环境

### 基本工具

```bash
$ yum install wget jq psmisc lrzsz vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git expect -y
```

### 安全配置

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service
```

### 同步时间

```bash
$ rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
$ yum install ntpdate -y
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
$ echo 'Asia/Shanghai' >/etc/timezone
$ ntpdate time2.aliyun.com
$ crontab -e
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com
```

### 配置源文件

```bash
$ rpm -ivh https://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/ceph-release-1-1.el7.noarch.rpm
```

### 配置免密

1）创建一个普通用户进行管理所有进程

ceph-deploy 在部署时需要以此普通用户的身份登录至各节点，可无密码使用 sudo 命令权限，并且在 SSH 时可免密钥登录

```bash
# 所有节点上创建用户并设置密码
$ useradd cephadmin && echo "cephadmin" | passwd cephadmin --stdin
```

2）在 administrator 节点生成并分发密钥（此操作需要在 cephadmin 用户下进行）

```bash
ssh-keygen -t rsa -P "" 
for i in mw-01 mw-02 mw-03 mw-04;do
expect -c "
spawn ssh-copy-id -i /home/cephadmin/.ssh/id_rsa.pub cephadmin@$i
        expect {
                \"*yes/no*\" {send \"yes\r\"; exp_continue}
                \"*password*\" {send \"cephadmin\r\"; exp_continue}
                \"*Password*\" {send \"cephadmin\r\";}
        } "
done
```

3）配置 sudo 权限（此操作需要在 root 用户下进行）

```bash
$ echo "cephadmin ALL = (root) NOPASSWD: ALL" | tee /etc/sudoers.d/cephadmin
$ chmod 0440 /etc/sudoers.d/cephadmin
```

## 安装 Ceph-deploy

在 administrator 节点上执行即可，主要是安装 ceph-deploy 软件包及其依赖包

```bash
$ yum install ceph-deploy python-setuptools python2-subprocess32
```

## 部署集群

Ceph-deploy 部署过程中会生成一些集群初始化配置文件和 key，后续扩容的时候也需要使用到。因此，建议在 administrator 节点上创建一个单独的目录，后续操作都需要使用 cephadmin 用户，进入到该目录中进行操作。

1）创建配置文件目录

> ceph-deploy 命令的操作

```bash
$ mkdir ceph-cluster && cd !$
```

2）创建一个 Ceph Cluster 集群，指定 cluster-network（集群内部通讯）和 public-network（外部访问Ceph集群）

```bash
$ ceph-deploy new --cluster-network 188.188.4.0/24 --public-network 188.188.4.0/24 mw-02
```

**部署命令详解**

- `new`：为创建一个新的集群，会有当前目录生成集群配置文件与 keyring 文件
- `--cluster-network`：指定集群内部使用网络
- `--public-network`：指定集群公用网络，也就是业务使用网络
- `mw-02`：指定初始的集群的第一个 monitor 节点的主机名，网络必可达


通过输出的信息可以看到，new 初始化集群过程中会生成 `ssh key` 密钥，`ceph.conf` 配置文件，`ceph.mon.keyring` 认证管理密钥

```bash
ll
total 204
-rw-rw-r--. 1 cephadmin cephadmin    261 Sep 15 17:28 ceph.conf              # 配置文件
-rw-rw-r--. 1 cephadmin cephadmin 198012 Sep 15 17:57 ceph-deploy-ceph.log   # 部署日志文件
-rw-------. 1 cephadmin cephadmin     73 Sep 15 17:28 ceph.mon.keyring       # monitor认证key

$ cat ceph.conf 
[global]
fsid = 68a5d089-b17d-4576-b9cd-ce5a91439001
public_network = 188.188.4.0/24
cluster_network = 188.188.4.0/24
mon_initial_members = mw-02
mon_host = 188.188.4.202
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```

```bash
先初始化monitor节点
$ ceph-deploy install mw-02 mw-03 mw-04

再初始化storage节点

手动方法
rpm -ivh https://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/ceph-release-1-1.el7.noarch.rpm
yum install ceph ceph-radosgw

$ ceph-deploy mon create-initial
```