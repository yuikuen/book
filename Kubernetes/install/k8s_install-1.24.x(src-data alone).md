# K8s Install(SRC_1.24.x-Data Alone)

> Kubernetes 二进制安装高可用集群(1.24.x) + Etcd(Alone)
>
> PS：安装前必须提前查看 [K8s-1.24.x 官方说明](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md#deprecation)，内有弃用参数说明。另有 [原文](https://sysdig.com/blog/kubernetes-1-24-whats-new/)/[译文](https://zhuanlan.zhihu.com/p/509074307) 参考

## 版本环境

**程序版本**

安装时请注意版本控制，如安装失败可检查各程序版本之间是否有兼容问题

- System：CentOS7.9.2009 Minimal
- Runtime：cri-containerd-cni-1.6.8
- Server：v1.24.x
- Etcd：v3.5.4
- Calico：v3.23.3
- CoreDNS：1.8.6
- Metricd：v0.6.1
- Dashboard：v2.6.0 & v1.0.8

**安装规划**

|HostName|ServerIP|Module|
|--|--|--|
|dglocal-1884141-datastorage01|188.188.4.141|Etcd|
|dglocal-1884142-datastorage02|188.188.4.142|Etcd|
|dglocal-1884143-datastorage03|188.188.4.143|Etcd|
|dglocal-1884150-k8smaster-lb|188.188.4.150|VIP_IP|
|dglocal-1884151-k8smaster01|188.188.4.151|kube{-apiserver,-controller-manager,-scheduler,let,-proxy},containerd|
|dglocal-1884152-k8smaster02|188.188.4.152|kube{-apiserver,-controller-manager,-scheduler,let,-proxy},containerd|
|dglocal-1884153-k8smaster03|188.188.4.153|kube{-apiserver,-controller-manager,-scheduler,let,-proxy},containerd|
|dglocal-1884161-k8snode01|188.188.4.161|kubelet,kube-proxy,containerd|
|dglocal-1884162-k8snode02|188.188.4.162|kubelet,kube-proxy,containerd|

**网络规划**

- Host-IP：188.188.4.0/24
- Pod-IP：172.10.0.0/12
- Server-IP：10.96.0.0/16

注：宿主机网段与 K8s 的 Server/Pod 网段不能重复

## 环境配置

> **安装说明：如未注明是在哪个节点执行，默认为全部节点都需要执行操作**

### 基本配置

1）安装常用工具

```bash
$ yum install wget jq psmisc lrzsz vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git expect -y
```

2）配置 Hostname & Hosts

```bash
# 设置各个节点的hostname
$ hostnamectl set-hostname dglocal-1884141-datastorage01 && bash
$ hostnamectl set-hostname dglocal-1884142-datastorage02 && bash
$ hostnamectl set-hostname dglocal-1884143-datastorage03 && bash
$ hostnamectl set-hostname dglocal-1884151-k8smaster01 && bash
$ hostnamectl set-hostname dglocal-1884152-k8smaster02 && bash
$ hostnamectl set-hostname dglocal-1884153-k8smaster03 && bash
$ hostnamectl set-hostname dglocal-1884161-k8snode01 && bash
$ hostnamectl set-hostname dglocal-1884162-k8snode02 && bash

# 所有节点配置hosts文件
$ cat >> /etc/hosts << EOF
188.188.4.141 dglocal-1884141-datastorage01
188.188.4.142 dglocal-1884142-datastorage02
188.188.4.143 dglocal-1884143-datastorage03
188.188.4.150 dglocal-1884150-k8smaster-lb
188.188.4.151 dglocal-1884151-k8smaster01
188.188.4.152 dglocal-1884152-k8smaster02
188.188.4.153 dglocal-1884153-k8smaster03
188.188.4.161 dglocal-1884161-k8snode01
188.188.4.162 dglocal-1884162-k8snode02
EOF
```

3）配置阿里云源

```bash
$ curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
$ yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
$ sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

4）关闭 Firewalld & SELINUX & Swap

```bash
$ systemctl disable --now firewalld NetworkManager

$ setenforce 0
$ sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/sysconfig/selinux
$ sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config

$ swapoff -a && sysctl -w vm.swappiness=0
$ sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

5）安装 ntpdate，同步时间

```bash
$ rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
$ yum install ntpdate -y
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
$ echo 'Asia/Shanghai' >/etc/timezone
$ ntpdate time2.aliyun.com
$ crontab -e
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com
```

6）设置 `limit` 最大打开文件数和最大进程数

```bash
$ ulimit -SHn 65535
$ vim /etc/security/limits.conf
# 末尾添加如下内容
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
```

7）配置密钥，实现免密登录

> 所有节点操作，互发送至每个节点

```bash
$ ssh-keygen -t rsa -P "" -f /root/.ssh/id_rsa
$ DataNodes='dglocal-1884141-datastorage01 dglocal-1884142-datastorage02 dglocal-1884143-datastorage03'
$ MasterNodes='dglocal-1884151-k8smaster01 dglocal-1884152-k8smaster02 dglocal-1884153-k8smaster03'
$ WorkNodes='dglocal-1884161-k8snode01 dglocal-1884162-k8snode02'
$ for NODE in $DataNodes $MasterNodes $WorkNodes;do
expect -c "
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@$NODE
        expect {
                \"*yes/no*\" {send \"yes\r\"; exp_continue}
                \"*password*\" {send \"password\r\"; exp_continue}
                \"*Password*\" {send \"password\r\";}
        } "
done 
```

### 内核升级

> 新版本使用的是 IPVS 模块，需要内核系统版本支持，建议提前升级内核

1）到 [ELREPO](https://elrepo.org/linux/kernel/) 下载自身需要的 RPM 包

```bash
# all nodes
$ mkdir /opt/software/kernel -p

# Master01下载
$ wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-ml-5.18.14-1.el7.elrepo.x86_64.rpm
$ wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-ml-devel-5.18.14-1.el7.elrepo.x86_64.rpm

$ DataNodes='dglocal-1884141-datastorage01 dglocal-1884142-datastorage02 dglocal-1884143-datastorage03'
$ MasterNodes='dglocal-1884152-k8smaster02 dglocal-1884153-k8smaster03'
$ WorkNodes='dglocal-1884161-k8snode01 dglocal-1884162-k8snode02'
$ for NODE in $DataNodes $MasterNodes $WorkNodes; do 
  echo $NODE; 
  scp -r /opt/software/kernel/* $NODE:/opt/software/kernel/;
done
```

2）直接加载安装

```bash
$ rpm -ivh kernel-ml-5.18.14-1.el7.elrepo.x86_64.rpm kernel-ml-devel-5.18.14-1.el7.elrepo.x86_64.rpm
warning: kernel-ml-5.18.14-1.el7.elrepo.x86_64.rpm: Header V4 DSA/SHA256 Signature, key ID baadae52: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:kernel-ml-devel-5.18.14-1.el7.elr################################# [ 50%]
   2:kernel-ml-5.18.14-1.el7.elrepo   ################################# [100%]
```

3）检查现有内核并重置启动项

```bash
$ awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
0 : CentOS Linux (5.18.14-1.el7.elrepo.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1160.el7.x86_64) 7 (Core)
2 : CentOS Linux (0-rescue-77ada07bc97843b3a8e034039509ee14) 7 (Core)

$ grub2-set-default 0
$ grub2-mkconfig -o /boot/grub2/grub.cfg && reboot now
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.18.14-1.el7.elrepo.x86_64
Found initrd image: /boot/initramfs-5.18.14-1.el7.elrepo.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-1160.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1160.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-4d7bc3c723f2437d995ca26258759d4e
Found initrd image: /boot/initramfs-0-rescue-4d7bc3c723f2437d995ca26258759d4e.img
done
```

4）安装 ipvsadm，配置模块

> 在内核 4.19+ 版本 nf_conntrack_ipv4 已经改为 nf_conntrack， 4.18 以下使用nf_conntrack_ipv4 即可

```bash
$ yum install ipvsadm ipset sysstat conntrack libseccomp -y
$ vim /etc/modules-load.d/ipvs.conf 
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip

$ systemctl enable --now systemd-modules-load.service
```

5）开启 k8s 集群中必须的内核参数

```bash
$ cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
net.ipv4.conf.all.route_localnet = 1

vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF

# 配置完内核后，重启服务器，检查以保证正常加载
$ sysctl --system && reboot now
$ lsmod | grep --color=auto -e ip_vs -e nf_conntrack
```

## 证书工具

1）下载部署需要的配置文件及自签证书工具

```bash
# Master01上执行操作
$ cd /root/; git clone https://github.91chi.fun/https://github.com/yuikuen/k8s-install.git
$ cd /root/k8s-install && git checkout k8s-install_v1.24.x

$ wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O /usr/local/bin/cfssl
$ wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson
$ chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
```

2）提前创建相关程序目录

```bash
# 所有Node节点上执行
$ mkdir -p /opt/cni/bin /etc/{etcd/ssl,kubernetes/{pki,manifests}} /var/{lib/kubelet,log/kubernetes} /etc/systemd/system/kubelet.service.d
```

## ETCD 数据库

> Etcd 数据库-单独部署集群，K8s 集群调用服务

1）创建证书 & 下载程序并解压分发至各节点

```bash
# Master01上执行操作
$ cd /root/k8s-install/pki
$ cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca
$ cfssl gencert \
  -ca=/etc/etcd/ssl/etcd-ca.pem \
  -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
  -config=ca-config.json \
  -hostname=127.0.0.1,dglocal-1884141-datastorage01,dglocal-1884142-datastorage02,dglocal-1884143-datastorage03,188.188.4.141,188.188.4.142,188.188.4.143 \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd

$ for i in dglocal-1884141-datastorage01; do echo $i;
  ssh $i "
  wget http://188.188.4.254:81/tools/Linux/Etcd/etcd-v3.5.4-linux-amd64.tar.gz -P /tmp &&
  tar -zxvf /tmp/etcd-v3.5.4-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.5.4-linux-amd64/etcd{,ctl} &&
  rm -rf /tmp/etcd*
  ";
  for d in dglocal-1884141-datastorage01 dglocal-1884142-datastorage02 dglocal-1884143-datastorage03; do echo $d;
    ssh $d "
    mkdir -p /etc/etcd/ssl &&
    mkdir -p /etc/kubernetes/pki/etcd
    ";
    for FILE in etcd-ca-key.pem etcd-ca.pem etcd-key.pem etcd.pem; do
      scp /etc/etcd/ssl/${FILE} $d:/etc/etcd/ssl/${FILE}
      scp $i:/usr/local/bin/* $d:/usr/local/bin/;
      done
    done
  done
```

2）连至各数据服务器创建配置文件

- Etcd Data01

```yaml
$ ssh dglocal-1884141-datastorage01
$ cat /etc/etcd/etcd.config.yml
name: 'dglocal-1884141-datastorage01'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://188.188.4.141:2380'
listen-client-urls: 'https://188.188.4.141:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://188.188.4.141:2380'
advertise-client-urls: 'https://188.188.4.141:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'dglocal-1884141-datastorage01=https://188.188.4.141:2380,dglocal-1884142-datastorage02=https://188.188.4.142:2380,dglocal-1884143-datastorage03=https://188.188.4.143:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

- Etcd Data02

```yaml
$ ssh dglocal-1884142-datastorage02
$ cat /etc/etcd/etcd.config.yml
name: 'dglocal-1884142-datastorage02'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://188.188.4.142:2380'
listen-client-urls: 'https://188.188.4.142:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://188.188.4.142:2380'
advertise-client-urls: 'https://188.188.4.142:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'dglocal-1884141-datastorage01=https://188.188.4.141:2380,dglocal-1884142-datastorage02=https://188.188.4.142:2380,dglocal-1884143-datastorage03=https://188.188.4.143:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

- Etcd Data03

```yaml
$ ssh dglocal-1884143-datastorage03
$ cat /etc/etcd/etcd.config.yml
name: 'dglocal-1884143-datastorage03'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://188.188.4.143:2380'
listen-client-urls: 'https://188.188.4.143:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://188.188.4.143:2380'
advertise-client-urls: 'https://188.188.4.143:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'dglocal-1884141-datastorage01=https://188.188.4.141:2380,dglocal-1884142-datastorage02=https://188.188.4.142:2380,dglocal-1884143-datastorage03=https://188.188.4.143:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

3）创建启动服务并检查数据状态

```bash
$ vim /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Service
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.config.yml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd3.service


$ ln -s /etc/etcd/ssl/* /etc/kubernetes/pki/etcd/
$ systemctl daemon-reload && systemctl enable --now etcd

$ export ETCDCTL_API=3
$ etcdctl --endpoints="188.188.4.143:2379,188.188.4.142:2379,188.188.4.141:2379" \
  --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem \
  --cert=/etc/kubernetes/pki/etcd/etcd.pem \
  --key=/etc/kubernetes/pki/etcd/etcd-key.pem \
  endpoint status --write-out=table
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 188.188.4.143:2379 | 6ff5229b3da78f44 |   3.5.4 |  9.8 MB |     false |      false |         3 |      37786 |              37786 |        |
| 188.188.4.142:2379 | 53bb7c8cf889dc1e |   3.5.4 |  9.8 MB |     false |      false |         3 |      37786 |              37786 |        |
| 188.188.4.141:2379 | b254a8ee397df155 |   3.5.4 |  9.8 MB |      true |      false |         3 |      37786 |              37786 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

## 基本组件

> 安装前需要优先升级 libseccomp，centos7 中 yum 下载的版本是 2.3，需要 2.4 以上
> （所有 Node 节点上操作）

1）查看 libseccomp 版本并作更新

```bash
$ rpm -qa | grep libseccomp
libseccomp-2.3.1-4.el7.x86_64
$ rpm -e libseccomp-* --nodeps
```

2）下载新版本 `libseccomp rpm` 安装

```bash
$ wget http://rpmfind.net/linux/centos/8-stream/BaseOS/x86_64/os/Packages/libseccomp-2.5.1-1.el8.x86_64.rpm
$ rpm -ivh libseccomp-2.5.1-1.el8.x86_64.rpm 
$ rpm -qa | grep libseccomp
libseccomp-2.5.1-1.el8.x86_64
```

3）配置 Containerd 所需的模块、内核设置

```bash
$ cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
$ systemctl restart systemd-modules-load.service

$ modprobe -- overlay
$ modprobe -- br_netfilter

$ cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
$ sysctl --system
```

4）下载并解压程序包

> cri-containerd-1.6.8-linux-amd64.tar.gz 包含containerd以及cri runc等相关工具包，建议下载本包

```bash
$ wget https://github.com/containerd/containerd/releases/download/v1.6.8/cri-containerd-1.6.8-linux-amd64.tar.gz
$ tar -zxvf cri-containerd-cni-1.6.8-linux-amd64.tar.gz -C /
```

5）创建 Containerd 配置文件

```bash
$ mkdir -p /etc/containerd 
$ containerd config default > /etc/containerd/config.toml

# 替换默认pause镜像地址
$ sed -i 's#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/google_containers#g' /etc/containerd/config.toml

# 配置systemd作为容器的cgroup driver
$ sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml

# 验证是否已修改
$ cat /etc/containerd/config.toml | grep -i sandbox_image
    sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
$ cat /etc/containerd/config.toml | grep -i SystemdCgroup
            SystemdCgroup = true
```

6）启动 systemd 服务

```bash
# 默认cri-containerd-cni包中已有containerd启动脚本，可直接调用启动
$ systemctl daemon-reload && systemctl enable --now containerd
```

7）配置 `crictl` 客户端连接运行时

```bash
$ cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
$ systemctl restart containerd
```

8）验证是否成功

```bash
$ crictl info
$ ctr version
Client:
  Version:  v1.6.8
  Revision: 9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
  Go version: go1.17.13

Server:
  Version:  v1.6.8
  Revision: 9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
  UUID: 0a71d5d8-34ed-4f6a-b8c6-9da1d48262c7
$ containerd --version
containerd github.com/containerd/containerd v1.6.8 9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
```

9）配置镜像加速

> 此为示例文件，仅供参考！建议提前下载对应镜像或采用本地私有仓库！
> 
> `k8s.gcr.io` 只是名称，可自定义修改，实际是从 `endpoint` 地址中拉取镜像

```bash
$ cat /etc/containerd/config.toml | grep -i registry.mirrors -A 1
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://eihzr0te.mirror.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://registry.cn-hongkong.aliyuncs.com/yuikuen/"]     
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
          endpoint = ["https://registry.cn-hongkong.aliyuncs.com"]
          
$ systemctl daemon-reload && systemctl restart containerd
```

注：配置镜像加速后需要重启服务

## 高可用配置

> 1、如不是高可用集群，HAProxy & KeepAlived 无需安装
> 
> 2、如使用云上托管服务，可直接使用云上 lb，如阿里云 slb、腾讯云 elb 等

所有 Master 节点进行安装

```bash
$ yum -y install haproxy keepalived
```

### HAProxy

> 所有 Master 都执行，配置均一样

```bash
$ vim /etc/haproxy/haproxy.cfg
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

frontend k8s-master
  bind 0.0.0.0:8443
  bind 127.0.0.1:8443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master

backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server dglocal-1884151-k8smaster01    188.188.4.151:6443  check
  server dglocal-1884152-k8smaster02    188.188.4.152:6443  check
  server dglocal-1884153-k8smaster03    188.188.4.153:6443  check
```

### KeepAlived

> 所有 Master 节点的 KeepAlived 配置不一样，注意每个节点的IP/网卡(interface参数)/virtual_router_id

```sh
# Master01
$ vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5 
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    mcast_src_ip 188.188.4.151
    virtual_router_id 150
    priority 101
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        188.188.4.150
    }
    track_script {
      chk_apiserver 
} }
```

```sh
# Master02
$ vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5 
    weight -5
    fall 2
    rise 1
 
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    mcast_src_ip 188.188.4.152
    virtual_router_id 150
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        188.188.4.150
    }
    track_script {
      chk_apiserver 
} }
```

```bash
# Master03
$ vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
    rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    mcast_src_ip 188.188.4.153
    virtual_router_id 150
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        188.188.4.150
    }
    track_script {
      chk_apiserver 
} }
```

### Check-UP

```bash
# 所有Master节点
$ vim /etc/keepalived/check_apiserver.sh 
#!/bin/bash
err=0
for k in $(seq 1 3)
do
    check_code=$(pgrep haproxy)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi

$ chmod +x /etc/keepalived/check_apiserver.sh
$ systemctl daemon-reload && systemctl enable --now haproxy keepalived
$ systemctl status haproxy keepalived
```

注：安装 HAProxy & KeepAlived 后，需要测试 VIP 是否正常生效

## 系统组件

以下操作均在 MasterNode 上操作，注意 `hostname`/`ServerIP`/`File_path`

1）下载 K8s 版本程序文件

```bash
$ wget https://dl.k8s.io/v1.24.4/kubernetes-server-linux-amd64.tar.gz
$ tar -xf kubernetes-server-linux-amd64.tar.gz --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}
$ kubelet --version
Kubernetes v1.24.4
```

2）解压并下发至其它节点

```bash
$ MasterNodes='k8sm02-4112-mgr k8sm03-4113-mgr'
$ WorkNodes='k8sn01-4114-app k8sn02-4115-app'
$ for NODE in $MasterNodes; do echo $NODE; 
  scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/; 
  scp /usr/local/bin/etcd* $NODE:/usr/local/bin/; 
 done
$ for NODE in $WorkNodes; do     
  scp /usr/local/bin/kube{let,-proxy} $NODE:/usr/local/bin/ ; 
 done
```

### Certificat

> 生成各组件模块的证书

```bash
$ cd /root/k8s-install/pki
$ cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca
```

- ApiServer

```sh
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/ca.pem \
  -ca-key=/etc/kubernetes/pki/ca-key.pem \
  -config=ca-config.json \
  -hostname=10.96.0.1,188.188.4.150,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,188.188.4.151,188.188.4.152,188.188.4.153 \
  -profile=kubernetes \
  apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver
```

- Front-proxy-client

```bash
$ cfssl gencert -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca 
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/front-proxy-ca.pem \
  -ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem \
  -config=ca-config.json -profile=kubernetes \
  front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client
```

- Controller-manager

```sh
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/ca.pem \
  -ca-key=/etc/kubernetes/pki/ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  manager-csr.json | cfssljson -bare /etc/kubernetes/pki/controller-manager
  
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.150:8443 \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
  
$ kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=/etc/kubernetes/pki/controller-manager.pem \
  --client-key=/etc/kubernetes/pki/controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
  
$ kubectl config set-context system:kube-controller-manager@kubernetes \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
  
$ kubectl config use-context system:kube-controller-manager@kubernetes \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
```

- Scheduler

```sh
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/ca.pem \
  -ca-key=/etc/kubernetes/pki/ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  scheduler-csr.json | cfssljson -bare /etc/kubernetes/pki/scheduler

$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.150:8443 \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

$ kubectl config set-credentials system:kube-scheduler \
  --client-certificate=/etc/kubernetes/pki/scheduler.pem \
  --client-key=/etc/kubernetes/pki/scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

$ kubectl config set-context system:kube-scheduler@kubernetes \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
  
$ kubectl config use-context system:kube-scheduler@kubernetes \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
```

- Admin

```sh
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/ca.pem \
  -ca-key=/etc/kubernetes/pki/ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare /etc/kubernetes/pki/admin

$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.150:8443 \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig
  
$ kubectl config set-credentials kubernetes-admin \
  --client-certificate=/etc/kubernetes/pki/admin.pem \
  --client-key=/etc/kubernetes/pki/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig
  
$ kubectl config set-context kubernetes-admin@kubernetes \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig
    
$ kubectl config use-context kubernetes-admin@kubernetes \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig
```

创建 `Secret`，并将其发至其它节点

```sh
$ openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
$ openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub

$ for NODE in dglocal-1884152-k8smaster02 dglocal-1884153-k8smaster03; do echo $NODE;
  for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); do 
    scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE};
 done; 
  for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig; do 
    scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE};
 done;
  for FILE in etcd-ca-key.pem etcd-ca.pem etcd-key.pem etcd.pem; do
    scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE};
 done;
done

# 查看所有Master节点
$ ls /etc/kubernetes/pki/ |wc -l
23
```

### ApiServer

> 所有 Master 节点创建 kube-apiserver service，注意如不是高可用集群，188.188.4.150 改为 Master01 地址

PS：1.24.x Kube-apiserver: 不安全的地址标志 `--address`,`--insecure-bind-address` 和 `--port( --insecure-portinert since 1.20)` 被移除

- insecure-port 的参数在最新 cdk 已经被注释了，这个参数在 K8s 1.24 会直接弃用
- [feature-gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)=LegacyServiceAccountTokenNoAutoGeneration：停止自动生成基于 Secret 的 服务帐户令牌

1）创建 Server 配置文件

- Master01

```bash
$ vim /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --v=2  \
  --logtostderr=true  \
  --allow-privileged=true  \
  --bind-address=0.0.0.0  \
  --secure-port=6443  \
  --advertise-address=188.188.4.151 \
  --service-cluster-ip-range=10.96.0.0/16  \
  --service-node-port-range=30000-32767  \
  --etcd-servers=https://188.188.4.141:2379,https://188.188.4.142:2379,https://188.188.4.143:2379 \
  --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
  --client-ca-file=/etc/kubernetes/pki/ca.pem  \
  --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
  --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
  --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
  --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
  --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
  --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
  --authorization-mode=Node,RBAC  \
  --enable-bootstrap-token-auth=true  \
  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
  --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
  --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
  --requestheader-allowed-names=aggregator  \
  --requestheader-group-headers=X-Remote-Group  \
  --requestheader-extra-headers-prefix=X-Remote-Extra-  \
  --requestheader-username-headers=X-Remote-User
  # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

- Master02

```bash
$ vim /usr/lib/systemd/system/kube-apiserver.service 
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --v=2  \
  --logtostderr=true  \
  --allow-privileged=true  \
  --bind-address=0.0.0.0  \
  --secure-port=6443  \
  --advertise-address=188.188.4.152 \
  --service-cluster-ip-range=10.96.0.0/16  \
  --service-node-port-range=30000-32767  \
  --etcd-servers=https://188.188.4.141:2379,https://188.188.4.142:2379,https://188.188.4.143:2379 \
  --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
  --client-ca-file=/etc/kubernetes/pki/ca.pem  \
  --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
  --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
  --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
  --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
  --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
  --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
  --authorization-mode=Node,RBAC  \
  --enable-bootstrap-token-auth=true  \
  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
  --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
  --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
  --requestheader-allowed-names=aggregator  \
  --requestheader-group-headers=X-Remote-Group  \
  --requestheader-extra-headers-prefix=X-Remote-Extra-  \
  --requestheader-username-headers=X-Remote-User
  # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

- Master03

```bash
$ vim /usr/lib/systemd/system/kube-apiserver.service 
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --v=2  \
  --logtostderr=true  \
  --allow-privileged=true  \
  --bind-address=0.0.0.0  \
  --secure-port=6443  \
  --advertise-address=188.188.4.153 \
  --service-cluster-ip-range=10.96.0.0/16  \
  --service-node-port-range=30000-32767  \
  --etcd-servers=https://188.188.4.141:2379,https://188.188.4.142:2379,https://188.188.4.143:2379 \
  --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
  --client-ca-file=/etc/kubernetes/pki/ca.pem  \
  --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
  --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
  --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
  --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
  --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
  --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
  --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
  --authorization-mode=Node,RBAC  \
  --enable-bootstrap-token-auth=true  \
  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
  --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
  --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
  --requestheader-allowed-names=aggregator  \
  --requestheader-group-headers=X-Remote-Group  \
  --requestheader-extra-headers-prefix=X-Remote-Extra-  \
  --requestheader-username-headers=X-Remote-User
  # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

2）所有 Master 节点开启服务并检查状态

```bash
$ systemctl daemon-reload && systemctl enable --now kube-apiserver
$ systemctl status kube-apiserver
```

### ControllerManager

> 所有 Master 节点的 kube-controller-manager service 配置一样

PS：自 v1.20 以来，不安全的地址标志 `--address` 和 `--port` kube-controller-manager 中的标志无效，并在 v1.24 中被删除

1）创建启动的服务文件

```bash
$ vim /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --v=2 \
  --logtostderr=true \
  --root-ca-file=/etc/kubernetes/pki/ca.pem \
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \
  --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \
  --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
  --leader-elect=true \
  --use-service-account-credentials=true \
  --node-monitor-grace-period=40s \
  --node-monitor-period=5s \
  --pod-eviction-timeout=2m0s \
  --controllers=*,bootstrapsigner,tokencleaner \
  --allocate-node-cidrs=true \
  --cluster-cidr=172.15.0.0/12 \
  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem \
  --node-cidr-mask-size=24
      
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

2）所有 Master 节点开启服务并检查状态

```bash
$ systemctl daemon-reload && systemctl enable --now kube-controller-manager
$ systemctl status kube-controller-manager
```

###  Scheduler

> 所有 Master 节点的 kube-scheduler service 配置一样

1）创建启动的服务文件

PS：Kube-scheduler 删除不安全的标志。可以改用 `--bind-address` 和 `--secure-port`

```bash
$ vim /usr/lib/systemd/system/kube-scheduler.service 
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --v=2 \
  --logtostderr=true \
  --leader-elect=true \
  --authentication-kubeconfig=/etc/kubernetes/scheduler.kubeconfig \
  --authorization-kubeconfig=/etc/kubernetes/scheduler.kubeconfig \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

2）所有 Master 节点开启服务并检查状态

```bash
$ systemctl daemon-reload && systemctl enable --now kube-scheduler
$ systemctl status kube-scheduler
```

## TLS Bootstrapping

> 注：如需修改文件中的 token-id 和 token-secret，需要保持一致

在配置文件中的 token 定义中，需要注意：

- token 的 name 必须是 `bootstrap-token-<token-id>` 的格式
- token 的 type 必须是 `bootstrap.kubernetes.io/token`
- token 的 `token-id` 和 `token-secret` 分别是6位和16位数字和字母的组合
- `auth-extra-groups` 定义了 token 代表的用户所属的额外的 group，而默认 group 名为 `system:bootstrappers`
- 这种类型 token 代表的用户名为 `system:bootstrap:<token-id>`，在本文中就是 `system:bootstrap:abcdef`

1）Master01 操作即可，生成随便字符串

```bash
$ cat /dev/urandom | head -n 22 | md5sum | head -c 22
dd4414.74c103b52e297e6c
$ cd /root/k8s-install/bootstrap
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.150:8443 \
  --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

$ kubectl config set-credentials tls-bootstrap-token-user \
  --token=dd4414.74c103b52e297e6c \
  --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

$ kubectl config set-context tls-bootstrap-token-user@kubernetes \
  --cluster=kubernetes \
  --user=tls-bootstrap-token-user \
  --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

$ kubectl config use-context tls-bootstrap-token-user@kubernetes \
  --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

$ mkdir -p /root/.kube ; cp /etc/kubernetes/admin.kubeconfig /root/.kube/config

# 确认前置配置状态
$ kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-1               Healthy   {"health":"true","reason":""}   
etcd-2               Healthy   {"health":"true","reason":""}   
etcd-0               Healthy   {"health":"true","reason":""} 

$ kubectl create -f bootstrap.secret.yaml
secret/bootstrap-token-dd4414 created
clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
clusterrolebinding.rbac.authorization.k8s.io/node-autoapprove-bootstrap created
clusterrolebinding.rbac.authorization.k8s.io/node-autoapprove-certificate-rotation created
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created
```

## WorkNodes

> Master & Node 均为工作节点 `WorkNode`，

下述操作在所有节点执行，首先将相关证书文件发送至各节点(新节点加入同理)

```bash
$ cd /etc/kubernetes/
$ MasterNodes='dglocal-1884152-k8smaster02 dglocal-1884153-k8smaster03'
$ WorkNodes='dglocal-1884161-k8snode01 dglocal-1884162-k8snode02'
$ for NODE in $MasterNodes $WorkNodes; do echo $NODE;
    for FILE in pki/ca.pem pki/ca-key.pem pki/front-proxy-ca.pem bootstrap-kubelet.kubeconfig; do
      scp /etc/kubernetes/$FILE $NODE:/etc/kubernetes/${FILE}
    done
  done
```

### Kubelet

1）所有节点配置 kubelet service

```bash
$ vim /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kubelet

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
```

2）所有节点创建 Kubelet 配置文件

> Runtime 为 Containerd，按以下内容创建 Kubelet 配置文件

PS：以下与 dockershim 相关的标志也将与 dockershim 一起删除 `--experimental-dockershim-root-directory, --docker-endpoint, --image-pull-progress-deadline, --network-plugin, --cni-conf-dir, --cni-bin-dir, --cni-cache-dir, --network-plugin-mtu`

```bash
$ vim /etc/systemd/system/kubelet.service.d/10-kubelet.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
Environment="KUBELET_CONFIG_ARGS=--config=/etc/kubernetes/kubelet-conf.yml"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node='' "
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_SYSTEM_ARGS $KUBELET_EXTRA_ARGS
---------------------------------------------------------------------------------
$ vim /etc/kubernetes/kubelet-conf.yml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
```

3）启动所有节点 kubelet 服务

```bash
$ systemctl daemon-reload && systemctl enable --now kubelet
$ systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubelet.conf
   Active: active (running) since Fri 2022-08-19 12:05:33 CST; 4min 29s ago
     Docs: https://github.com/kubernetes/kubernetes
 Main PID: 72194 (kubelet)
    Tasks: 18
   Memory: 41.2M
   CGroup: /system.slice/kubelet.service
           └─72194 /usr/local/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --config=/etc/kubernetes/kubelet-conf.yml --co...

Aug 19 12:05:33 k8s-master01 kubelet[72194]: I0819 12:05:33.688575   72194 state_mem.go:88] "Updated default CPUSet" cpuSet=""
Aug 19 12:05:33 k8s-master01 kubelet[72194]: I0819 12:05:33.688594   72194 state_mem.go:96] "Updated CPUSet assignments" assignments=map[]
...
```
注意：`Active: active (running)` 并不代表完全正常，信息输出 `E0819` + `Error ...` 同样是失败，需要状态为 `Active: active (running)` 并且输出信息都为 `I0819` 才算正常

4）可查看系统日志和集群状态

```bash
# 如果正常日志信息不会循环打印
$ tail -f /var/log/messages
$ kubectl get node
NAME                          STATUS   ROLES    AGE     VERSION
dglocal-1884151-k8smaster01   Ready    <none>   4h17m   v1.24.4
dglocal-1884152-k8smaster02   Ready    <none>   4h17m   v1.24.4
dglocal-1884153-k8smaster03   Ready    <none>   4h17m   v1.24.4
dglocal-1884161-k8snode01     Ready    <none>   4h17m   v1.24.4
dglocal-1884162-k8snode02     Ready    <none>   4h17m   v1.24.4
```

注：此处的 `Ready` 不代表 Node 连接正常，还需要配置 `Proxy/Calico/CoreDNS`

### Kube-Proxy

> 仅在 Master01 执行

1）通过 Master01 获取参数值并生成 `kubeconfig` 文件

```bash
$ cd /root/k8s-install
$ kubectl -n kube-system create serviceaccount kube-proxy
$ kubectl create clusterrolebinding system:kube-proxy \
  --clusterrole system:node-proxier \
  --serviceaccount kube-system:kube-proxy

$ SECRET=$(kubectl -n kube-system get sa/kube-proxy --output=jsonpath='{.secrets[0].name}')
$ JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET --output=jsonpath='{.data.token}' | base64 -d)

$ PKI_DIR=/etc/kubernetes/pki
$ K8S_DIR=/etc/kubernetes
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.150:8443 \
  --kubeconfig=${K8S_DIR}/kube-proxy.kubeconfig

$ kubectl config set-credentials kubernetes \
  --token=${JWT_TOKEN} \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

$ kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=kubernetes \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

$ kubectl config use-context kubernetes --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
```

2）将 `kube-proxy.kubeconfig` 发至其他节点

```bash
$ MasterNodes='dglocal-1884152-k8smaster02 dglocal-1884153-k8smaster03'
$ WorkNodes='dglocal-1884161-k8snode01 dglocal-1884162-k8snode02'
$ for NODE in $MasterNodes $WorkNodes; do echo $NODE;
    scp /etc/kubernetes/kube-proxy.kubeconfig  $NODE:/etc/kubernetes/kube-proxy.kubeconfig
  done
```

3）所有节点添加 kube-proxy 的配置和 service 文件

```bash
$ vim /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --v=2

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
---------------------------------------------------------------------------------
$ vim /etc/kubernetes/kube-proxy.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.15.0.0/12 
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms
```

4）所有节点启动 kube-proxy

```bash
$ systemctl daemon-reload && systemctl enable --now kube-proxy
$ systemctl status kube-proxy
```

## Calico

> Master01 节点执行即可
>
> 安装前请参考安装要求，根据数据存储和节点数量，选择参数文件来安装 Calico

- [Calico 安装要求](https://docs.projectcalico.org/getting-started/kubernetes/requirements)
- [Calico Github](https://github.com/projectcalico/calico)
- [Calico Yaml 参考文件](https://github.com/projectcalico/calico/tree/master/manifests)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220816180050.png)

1）下载程序文件

```bash
# 最新版本
$ curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
$ curl https://projectcalico.docs.tigera.io/manifests/calico-typha.yaml -o calico.yaml
$ curl https://projectcalico.docs.tigera.io/manifests/calico-etcd.yaml -o calico.yaml

# 选择推荐版本
$ curl https://projectcalico.docs.tigera.io/archive/v3.23/manifests/calico.yaml -O
$ curl https://projectcalico.docs.tigera.io/archive/v3.23/manifests/calico-typha.yaml -o calico.yaml
$ curl https://projectcalico.docs.tigera.io/archive/v3.23/manifests/calico-etcd.yaml -o calico.yaml
```

2）根据节点及需求选择版本安装

- **官方推荐版本**
  
```bash
$ curl https://projectcalico.docs.tigera.io/archive/v3.23/manifests/calico-typha.yaml -o calico.yaml

$ POD_SUBNET="172.15.0.0/12"
$ sed -i 's@# - name: CALICO_IPV4POOL_CIDR@- name: CALICO_IPV4POOL_CIDR@g; s@#   value: "192.168.0.0/16"@  value: '"${POD_SUBNET}"'@g' calico.yaml
$ grep "IPV4POOL_CIDR" calico.yaml -A 1
            - name: CALICO_IPV4POOL_CIDR
              value: "172.15.0.0/12"

$ kubectl apply -f calico.yaml
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
service/calico-typha created
deployment.apps/calico-typha created
poddisruptionbudget.policy/calico-typha created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
poddisruptionbudget.policy/calico-kube-controllers created

$ kubectl get po -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS       AGE
kube-system            calico-kube-controllers-6754f569c-2rd5f      1/1     Running   4 (135m ago)   4h15m
kube-system            calico-node-85tx2                            1/1     Running   2 (135m ago)   4h15m
kube-system            calico-node-96n9q                            1/1     Running   2 (135m ago)   4h15m
kube-system            calico-node-dwrkq                            1/1     Running   2 (135m ago)   4h15m
kube-system            calico-node-gg9h4                            1/1     Running   2 (135m ago)   4h15m
kube-system            calico-node-szlt6                            1/1     Running   2 (135m ago)   4h15m
kube-system            calico-typha-bb4f6c8b5-tfdvs                 1/1     Running   2 (135m ago)   4h15m

$ crictl pods
POD ID              CREATED             STATE               NAME                           NAMESPACE           ATTEMPT             RUNTIME
eea868a019521       2 hours ago         Ready               calico-node-gg9h4              kube-system         2                   (default)
f6cf0c2127d7c       2 hours ago         Ready               calico-typha-bb4f6c8b5-tfdvs   kube-system         2                   (default)
b7e8bd00ed826       4 hours ago         NotReady            calico-node-gg9h4              kube-system         1                   (default)
ff9cc515c391e       4 hours ago         NotReady            calico-typha-bb4f6c8b5-tfdvs   kube-system         1                   (default)

$ crictl images
IMAGE                                                          TAG                 IMAGE ID            SIZE
registry.cn-hangzhou.aliyuncs.com/google_containers/pause      3.6                 6270bb605e12e       302kB
registry.cn-hongkong.aliyuncs.com/yuikuen/cni                  v3.23.3             64a041d6e4236       77.9MB
registry.cn-hongkong.aliyuncs.com/yuikuen/node                 v3.23.3             3a6768785333d       71.4MB
registry.cn-hongkong.aliyuncs.com/yuikuen/pod2daemon-flexvol   v3.23.3             016bb666554ef       8.66MB
registry.cn-hongkong.aliyuncs.com/yuikuen/typha                v3.23.3             5c116e2d13d34       51.4MB
```

- **with etcd datastore**
  
  - 编辑 `kind: ConfigMap` 的 etcd_endpoints，添加3个 etcd 的 url 地址
  - 编辑 `kind: Secret` 中的内容，k8s 定义的证书 base64 的加密数据(etcd_ca、etcd_cert、etcd_key)，把此三个证书数据经过 base64 方式加密后，添加到 calico.yaml 中
  - 添加上述 etcd 证书的路径进行挂载
  - 编辑 `name: CALICO_IPV4POOL_CIDR` 添加 Pod IP 段(值与 `kube-contrller-manager` 的 systemd 配置文件 `--cluster-cidr` 或 `kubelet-conf.yaml` 的 `podCIDR` 一致 )

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220817100016.png)

```bash
$ curl https://projectcalico.docs.tigera.io/archive/v3.23/manifests/calico-etcd.yaml -o calico.yaml

$ sed -i 's#etcd_endpoints: "http://<ETCD_IP>:<ETCD_PORT>"#etcd_endpoints: "https://188.188.4.151:2379,https://188.188.4.152:2379,https://188.188.4.153:2379"#g' calico-etcd.yaml
$ ETCD_CA=`cat /etc/kubernetes/pki/etcd/etcd-ca.pem | base64 | tr -d '\n'`
$ ETCD_CERT=`cat /etc/kubernetes/pki/etcd/etcd.pem | base64 | tr -d '\n'`
$ ETCD_KEY=`cat /etc/kubernetes/pki/etcd/etcd-key.pem | base64 | tr -d '\n'`
$ sed -i "s@# etcd-key: null@etcd-key: ${ETCD_KEY}@g; s@# etcd-cert: null@etcd-cert: ${ETCD_CERT}@g; s@# etcd-ca: null@etcd-ca: ${ETCD_CA}@g" calico-etcd.yaml
$ sed -i 's#etcd_ca: ""#etcd_ca: "/calico-secrets/etcd-ca"#g; s#etcd_cert: ""#etcd_cert: "/calico-secrets/etcd-cert"#g; s#etcd_key: "" #etcd_key: "/calico-secrets/etcd-key" #g' calico-etcd.yaml
$ POD_SUBNET="172.15.0.0/12"
$ sed -i 's@# - name: CALICO_IPV4POOL_CIDR@- name: CALICO_IPV4POOL_CIDR@g; s@#   value: "192.168.0.0/16"@  value: '"${POD_SUBNET}"'@g' calico-etcd.yaml

$ kubectl apply -f calico.yaml
$ kubectl get po -n kube-system
```
注：如果容器状态异常可以使用 kubectl describe 或者 kubectl logs 查看容器的日志

## CoreDNS

> 安装前请参考 [CoreDNS GitHub](https://github.com/coredns/deployment/blob/master/kubernetes/CoreDNS-k8s_version.md) 版本介绍和 [yaml 示例文件](https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed)/[1.86 yaml示例文件](https://github.com/coredns/deployment/commit/8e13ce82cc607c53551dfe3ab40aef788a69c1ac)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220819173018.png)

- **官方推荐版本**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220817181324.png)

```bash
# 注意示例文件已提前修改过内容，主要修改REVERSE_CIDRS、DNS_DOMAIN、CLUSTER_DNS_IP等变量为实际值
$ cd /root/k8s-install/CoreDNS
$ COREDNS_SERVICE_IP=`kubectl get svc | grep kubernetes | awk '{print $3}'`0
$ echo $COREDNS_SERVICE_IP
10.96.0.10
$ sed -i "s#CLUSTER_DNS_IP#${COREDNS_SERVICE_IP}#g" coredns.yaml
$ grep "clusterIP" coredns.yaml 
  clusterIP: 10.96.0.10
  
$ kubectl create -f coredns.yaml 
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
$ kubectl get po -n kube-system -l k8s-app=kube-dns
NAME                      READY   STATUS    RESTARTS   AGE
coredns-5db5696c7-cj577   1/1     Running   0          14s
```

- **最新版本**

> 最新版本未经测试，建议选用推荐版本，具体更新如下图所示

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220819173356.png)

```bash
$ COREDNS_SERVICE_IP=`kubectl get svc | grep kubernetes | awk '{print $3}'`0
$ echo $COREDNS_SERVICE_IP
10.96.0.10

$ git clone https://github.com/coredns/deployment.git
$ cd deployment/kubernetes
$ ./deploy.sh -s -i ${COREDNS_SERVICE_IP} | kubectl apply -f -
$ kubectl get po -n kube-system -l k8s-app=kube-dns
```

## Metrics Server

> 在新版的 Kubernetes 中系统资源的采集均使用 Metrics-server，可以通过 Metrics 采集节点和 Pod 的内存、磁盘、CPU 和网络的使用率

**安装前可查看 [官方 GitHub](https://github.com/kubernetes-sigs/metrics-server/) 版本兼容是否合适**

| Metrics Server | Metrics API group/version | Supported Kubernetes version |
| -------------- | ------------------------- | ---------------------------- |
| 0.6.x          | `metrics.k8s.io/v1beta1`  | 1.19+                        |
| 0.5.x          | `metrics.k8s.io/v1beta1`  | *1.8+                        |
| 0.4.x          | `metrics.k8s.io/v1beta1`  | *1.8+                        |
| 0.3.x          | `metrics.k8s.io/v1beta1`  | 1.8-1.21                     |

注：提前下载镜像并修改镜像地址信息和添加证书认证，否则无法下载到镜像

- **官方推荐版本**

```bash
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.1/components.yaml
# 添加证书验证并修改证书
# nu:129~137
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        # 添加证书认证内容，参考'配置聚合层',并修改镜像地址
        - --kubelet-insecure-tls
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem # change to front-proxy-ca.crt for kubeadm
        - --requestheader-username-headers=X-Remote-User
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        image: registry.cn-hongkong.aliyuncs.com/yuikuen/metrics-server:v0.6.1

# nu:167~176
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
        # 添加证书路径，进行hostPath挂载
        - name: ca-ssl
          mountPath: /etc/kubernetes/pki
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
      - name: ca-ssl
        hostPath:
          path: /etc/kubernetes/pki
          
$ kubectl create -f .
$ kubectl get po -n kube-system -l k8s-app=metrics-server
NAME                              READY   STATUS    RESTARTS   AGE
metrics-server-589b87cf7c-p4dk6   1/1     Running   0          42s
```

- **最新版本**

> 同官方推荐操作方法一样，提前下载镜像并修改镜像地址信息和添加证书认证

```bash
# 修改方法一样，不再详细说明
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
$ kubectl create -f .
$ kubectl get po -n kube-system -l k8s-app=metrics-server
```

## Dashboard

> [Dashboard](https://github.com/kubernetes/dashboard) 用于展示集群中的各类资源，同时也可以实时查看 Pod 的日志和在容器中执行一些命令等

**安装前请查看 [Dashboard Releases](https://github.com/kubernetes/dashboard/releases) 的版本介绍，保证与 K8s 的兼容**

| Kubernetes version | 1.21 | 1.22 | 1.23 | 1.24 |
| ------------------ | ---- | ---- | ---- | ---- |
| Compatibility      | ?    | ?    | ?    | ✓    |

- `✓` Fully supported version range.
- `?` Due to breaking changes between Kubernetes API versions, some features might not work correctly in the Dashboard.

1）下载兼容的版本文件

```bash
# 官方推荐版本
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml

# 最新版本(不建议)
$ wget kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/recommended.yaml
```

2）创建管理员的 `Account` 文件

```bash
$ cat dashboard-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1 
kind: ClusterRoleBinding 
metadata: 
  name: admin-user
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
    license: 34C7357D6D1C641D4CA1E369E7244F61
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

3）浏览器访问(Chrome)需要添加启动参数，否则因安全问题无法访问

解决方法：浏览器快捷方式增加参数 `--test-type --ignore-certificate-errors`

4）更改 dashboard 的 svc 为 NodePort，暴露端口进行访问

```bash
$ kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
ports:
- port: 443
  protocol: TCP
  targetPort: 8443
  # 可自定义
  nodePort: 30000
selector:
  k8s-app: kubernetes-dashboard
sessionAffinity: None
# 将ClusterIP改为NodePort
type: NodePort

$ kubectl get po,svc -n kubernetes-dashboard
```

5）查看管理员的 token 值

```bash
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220818104441.png)