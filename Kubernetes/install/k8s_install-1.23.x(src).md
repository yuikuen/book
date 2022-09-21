# K8s Install(SRC_1.23.x)

> Kubernetes 二进制安装高可用 1.23.x

## 版本环境

**程序版本**

安装时请注意版本控制，如安装失败可检查各程序版本之间是否有兼容问题

- System：CentOS7.9.2009 Minimal
- Runtime：docker-ce-20.10.*
- Server：v1.23.x
- Etcd：v3.5.4
- Calico：v3.22.0
- CoreDNS：1.8.6
- Metricd：v0.5.2
- Dashboard：v2.5.0 & v1.0.7

**安装规划**

| hostname | ServerIP | Module |
|--|--|--|
|k8s-master-lb|188.188.4.120|KeepAlived IP(VIP)|
|k8s-master01~03|188.188.4.121~123|kube-apiserver,kube-controller-manager,kube-scheduler,etcd,kubelet,kube-proxy,docker/containerd|
|k8s-node01~02|188.188.4.124~125|kubelet,kube-proxy,docker/containerd|

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
$ hostnamectl set-hostname k8s-master01 && bash
$ hostnamectl set-hostname k8s-master02 && bash
$ hostnamectl set-hostname k8s-master03 && bash
$ hostnamectl set-hostname k8s-node01 && bash
$ hostnamectl set-hostname k8s-node02 && bash

$ cat >> /etc/hosts << EOF
188.188.4.121 k8s-master01  # 6C10G 40G
188.188.4.122 k8s-master02  # 6C10G 40G
188.188.4.123 k8s-master03  # 6C10G 40G
188.188.4.120 k8s-master-lb # VIP虚拟IP不占用机器资源
188.188.4.124 k8s-node01    # 4C10G 40G
188.188.4.125 k8s-node02    # 4C10G 40G
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

> Master01 节点操作，发送至其他节点

```bash
$ ssh-keygen -t rsa -P "" -f /root/.ssh/id_rsa
$ for i in k8s-master01 k8s-master02 k8s-master03 k8s-node01 k8s-node02;do
expect -c "
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@$i
        expect {
                \"*yes/no*\" {send \"yes\r\"; exp_continue}
                \"*password*\" {send \"passwd\r\"; exp_continue}
                \"*Password*\" {send \"passwd\r\";}
        } "
done 
```

### 内核升级

> 新版本使用的是 IPVS 模块，需要内核系统版本支持，建议提前升级内核

1）到 [ELREPO](https://elrepo.org/linux/kernel/) 下载自身需要的 RPM 包

```bash
$ wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-ml-5.18.14-1.el7.elrepo.x86_64.rpm
$ wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-ml-devel-5.18.14-1.el7.elrepo.x86_64.rpm
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

## 基本组件

> 所有节点都安装一样的基本组件

### Docker Runtime

```bash
$ yum list docker-ce --showduplicates | sort -r
$ yum install docker-ce-20.10.* docker-cli-20.10.* -y

$ mkdir -p /etc/docker/
$ cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

$ systemctl daemon-reload && systemctl enable --now docker && systemctl status docker
```

### Containerd Runtime

> 安装方法有两种，可根据自身需求进行

- Yum 方式安装

1）查看现有版本

```bash
$ yum list | grep containerd
containerd.io.x86_64                      1.6.7-3.1.el7                docker-ce-stable

# 安装docker时，同时安装containerd
$ yum install docker-ce-20.10.* docker-ce-cli-20.10.* containerd -y 
```

2）配置 Containerd 所需的模块、内核设置

```bash
$ cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

$ modprobe -- overlay
$ modprobe -- br_netfilter

$ cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
$ sysctl --system
```

3）生成 Containerd 的默认配置文件，并将 Cgroup 改为 Systemd，和修改镜像下载地址

```bash
$ mkdir -p /etc/containerd
$ containerd config default | tee /etc/containerd/config.toml

# 修改k8s.gcr.io/pause:3.6为registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6并修改SystemdCgroup为true
$ sed -i 's#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/google_containers#g' /etc/containerd/config.toml
$ sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml

# 验证是否已修改
$ cat /etc/containerd/config.toml | grep -i sandbox_image
  sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5"
$ cat /etc/containerd/config.toml | grep -i SystemdCgroup
  SystemdCgroup = true
```

4）启动服务并配置 crictl 客户端连接的运行时位置

```bash
$ systemctl daemon-reload && systemctl enable --now containerd

$ cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

$ ctr version
Client:
  Version:  1.6.7
  Revision: 3df54a852345ae127d1fa3092b95168e4a88e2f8
  Go version: go1.17.8

Server:
  Version:  1.6.7
  Revision: 3df54a852345ae127d1fa3092b95168e4a88e2f8
  UUID: d6ad552e-4bd8-498d-a4d1-ddedf391bc1b
```

- Source

> 安装前需要优先升级 libseccomp，centos7 中 yum 下载的版本是 2.3，需要 2.4 以上

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
$ containerd --version
```

### K8s & Etcd

> **注意：选用 K8s 版本前，可查看 K8s CHANGELOG 文件，对照 Etcd 版本**

- [Etcd GitHub 地址](https://github.com/etcd-io/etcd/releases)
- [K8s Server GitHub 地址](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220816172604.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220816172724.png)

1）提前创建程序目录

```bash
$ mkdir -p /opt/cni/bin /etc/{etcd/ssl,kubernetes/{pki,manifests}} /var/{lib/kubelet,log/kubernetes} /etc/systemd/system/kubelet.service.d
```

2）下载相关配置文件

```bash
# Master01操作即可
$ cd /root/; git clone https://github.com/yuikuen/k8s-install.git
$ cd /root/k8s-install && git checkout k8s-install_v1.23.x

$ wget https://dl.k8s.io/v1.23.9/kubernetes-server-linux-amd64.tar.gz
$ wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz
```

3）解压并发至其它节点

```bash
$ tar -xf kubernetes-server-linux-amd64.tar.gz --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}
$ tar -zxvf etcd-v3.5.4-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.5.4-linux-amd64/etcd{,ctl}

$ kubelet --version
Kubernetes v1.23.9

$ etcdctl version
etcdctl version: 3.5.4
API version: 3.5
```

```bash
$ MasterNodes='k8s-master02 k8s-master03'
$ WorkNodes='k8s-node01 k8s-node02'
$ for NODE in $MasterNodes; do echo $NODE; 
  scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/; 
  scp /usr/local/bin/etcd* $NODE:/usr/local/bin/; 
 done
$ for NODE in $WorkNodes; do     
  scp /usr/local/bin/kube{let,-proxy} $NODE:/usr/local/bin/ ; 
 done
```

## 生成证书

> 下载自签工具，生成证书

```bash
# Master01操作即可
$ wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O /usr/local/bin/cfssl
$ wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson
$ chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
```

### Etcd 证书

> Master01 节点生成 Etcd 证书，生成证书的 CSR 文件：证书签名请求文件，配置了一些域名、公司、单位

```bash
$ cd /root/k8s-install/pki

# 生成etcd CA证书和CA证书的key，注意hostname、profile参数要与证书(ca-config.json)设置对应
$ cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca
$ cfssl gencert \
  -ca=/etc/etcd/ssl/etcd-ca.pem \
  -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
  -config=ca-config.json \
  -hostname=127.0.0.1,k8s-master01,k8s-master02,k8s-master03,188.188.4.121,188.188.4.122,188.188.4.123 \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd
```

```bash
$ MasterNodes='k8s-master02 k8s-master03'
$ WorkNodes='k8s-node01 k8s-node02'
$ for NODE in $MasterNodes; do
    ssh $NODE "mkdir -p /etc/etcd/ssl"
     for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem; do
       scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE}
     done
 done
```

### K8s 组件证书

> 如不做高可用集群，10.96.0.1 将改成 Master01 的 IP

1）生成各组件证书

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca
```

- ApiServer

```bash
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/ca.pem \
  -ca-key=/etc/kubernetes/pki/ca-key.pem \
  -config=ca-config.json \
  -hostname=10.96.0.1,188.188.4.120,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,188.188.4.121,188.188.4.122,188.188.4.123 \
  -profile=kubernetes \
  apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver
```

- Front-proxy-client(生成 apiserver 的聚合证书)

```bash
$ cfssl gencert -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca 
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/front-proxy-ca.pem \
  -ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem \
  -config=ca-config.json -profile=kubernetes \
  front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client
```

- Controller-manage

```bash
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/ca.pem \
  -ca-key=/etc/kubernetes/pki/ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  manager-csr.json | cfssljson -bare /etc/kubernetes/pki/controller-manager
 
# set-cluster：设置一个集群项
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.120:8443 \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# set-credentials 设置一个用户项
$ kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=/etc/kubernetes/pki/controller-manager.pem \
  --client-key=/etc/kubernetes/pki/controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
  
# 设置一个环境项，一个上下文
$ kubectl config set-context system:kube-controller-manager@kubernetes \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 使用某个环境当做默认环境
$ kubectl config use-context system:kube-controller-manager@kubernetes \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
```

- Scheduler

```bash
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/ca.pem \
  -ca-key=/etc/kubernetes/pki/ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  scheduler-csr.json | cfssljson -bare /etc/kubernetes/pki/scheduler

# set-cluster：设置一个集群项
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.120:8443 \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

# set-credentials 设置一个用户项
$ kubectl config set-credentials system:kube-scheduler \
  --client-certificate=/etc/kubernetes/pki/scheduler.pem \
  --client-key=/etc/kubernetes/pki/scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

# 设置一个环境项，一个上下文
$ kubectl config set-context system:kube-scheduler@kubernetes \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
  
# 使用某个环境当做默认环境
$ kubectl config use-context system:kube-scheduler@kubernetes \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
```

- Admin

```bash
$ cfssl gencert \
  -ca=/etc/kubernetes/pki/ca.pem \
  -ca-key=/etc/kubernetes/pki/ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare /etc/kubernetes/pki/admin

# set-cluster：设置一个集群项
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.120:8443 \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig
  
# set-credentials 设置一个用户项
$ kubectl config set-credentials kubernetes-admin \
  --client-certificate=/etc/kubernetes/pki/admin.pem \
  --client-key=/etc/kubernetes/pki/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig
  
# 设置一个环境项，一个上下文
$ kubectl config set-context kubernetes-admin@kubernetes \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig
    
# 使用某个环境当做默认环境
$ kubectl config use-context kubernetes-admin@kubernetes \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig
```

2）创建 Secret，并将其发至其它节点

```bash
$ openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
$ openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub
```

```bash
$ for NODE in k8s-master02 k8s-master03; do 
  for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); do 
    scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE};
 done; 
  for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig; do 
    scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE};
 done;
done

# 查看证书文件
$ ls /etc/kubernetes/pki/ |wc -l
23
```

## 高可用配置

> 如不是高可用集群，HAProxy & KeepAlived 无需安装
>
> 如使用云上托管服务，可直接使用云上 lb，如阿里云 slb、腾讯云 elb 等

1）所有 Master 节点进行安装

```bash
$ yum -y install haproxy keepalived
```

### HAProxy

```bash
# 所有Master的HAProxy配置一样
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
  server k8s-master01    188.188.4.121:6443  check
  server k8s-master02    188.188.4.122:6443  check
  server k8s-master03    188.188.4.123:6443  check
```

### KeepAlived

> 所有 Master 节点的 KeepAlived 配置不一样，注意每个节点的IP和网卡（interface参数）

- Master01

```bash
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
    mcast_src_ip 188.188.4.121
    virtual_router_id 120
    priority 101
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        188.188.4.120
    }
    track_script {
      chk_apiserver 
} }
```

- Master02

```bash
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
    mcast_src_ip 188.188.4.122
    virtual_router_id 120
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        188.188.4.120
    }
    track_script {
      chk_apiserver 
} }
```

- Master03

```bash
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
    mcast_src_ip 188.188.4.123
    virtual_router_id 120
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        188.188.4.120
    }
    track_script {
      chk_apiserver 
} }
```

### 健康检查

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

# 授权并启动服务测试
$ chmod +x /etc/keepalived/check_apiserver.sh
$ systemctl daemon-reload && systemctl enable --now haproxy keepalived
$ systemctl status haproxy keepalived
```
注：安装 HAProxy & KeepAlived 后，需要测试 VIP 是否正常生效

## 组件配置

### Etcd

> 配置大致相同，注意修改每个 Master 节点的 etcd 配置的主机名和 IP 地址

1）所有 Master 节点创建配置文件

- Master01

```bash
$ vim /etc/etcd/etcd.config.yml
name: 'k8s-master01'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://188.188.4.121:2380'
listen-client-urls: 'https://188.188.4.121:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://188.188.4.121:2380'
advertise-client-urls: 'https://188.188.4.121:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://188.188.4.121:2380,k8s-master02=https://188.188.4.122:2380,k8s-master03=https://188.188.4.123:2380'
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

- Master02

```bash
$ vim /etc/etcd/etcd.config.yml
name: 'k8s-master02'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://188.188.4.122:2380'
listen-client-urls: 'https://188.188.4.122:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://188.188.4.122:2380'
advertise-client-urls: 'https://188.188.4.122:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://188.188.4.121:2380,k8s-master02=https://188.188.4.122:2380,k8s-master03=https://188.188.4.123:2380'
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

- Master03

```bash
$ vim /etc/etcd/etcd.config.yml
name: 'k8s-master03'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://188.188.4.123:2380'
listen-client-urls: 'https://188.188.4.123:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://188.188.4.123:2380'
advertise-client-urls: 'https://188.188.4.123:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://188.188.4.121:2380,k8s-master02=https://188.188.4.122:2380,k8s-master03=https://188.188.4.123:2380'
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

2）所有 Master 节点创建 Service 服务文件

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

# 创建etcd的证书目录
$ mkdir /etc/kubernetes/pki/etcd
$ ln -s /etc/etcd/ssl/* /etc/kubernetes/pki/etcd/
$ systemctl daemon-reload && systemctl enable --now etcd

# 查看Etcd状态
$ export ETCDCTL_API=3
$ etcdctl --endpoints="188.188.4.123:2379,188.188.4.122:2379,188.188.4.121:2379" \
  --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem \
  --cert=/etc/kubernetes/pki/etcd/etcd.pem \
  --key=/etc/kubernetes/pki/etcd/etcd-key.pem \
  endpoint status --write-out=table
```

### ApiServer

> 所有 Master 节点创建 kube-apiserver service，注意如不是高可用集群，188.188.4.120 改为 Master01 地址

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
  --insecure-port=0  \
  --advertise-address=188.188.4.121 \
  --service-cluster-ip-range=10.96.0.0/16  \
  --service-node-port-range=30000-32767  \
  --etcd-servers=https://188.188.4.121:2379,https://188.188.4.122:2379,https://188.188.4.123:2379 \
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
  --insecure-port=0  \
  --advertise-address=188.188.4.122 \
  --service-cluster-ip-range=10.96.0.0/16  \
  --service-node-port-range=30000-32767  \
  --etcd-servers=https://188.188.4.121:2379,https://188.188.4.122:2379,https://188.188.4.123:2379 \
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
  --insecure-port=0  \
  --advertise-address=188.188.4.123 \
  --service-cluster-ip-range=10.96.0.0/16  \
  --service-node-port-range=30000-32767  \
  --etcd-servers=https://188.188.4.121:2379,https://188.188.4.122:2379,https://188.188.4.123:2379 \
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

所有 Master 节点开启服务并检查状态

```bash
$ systemctl daemon-reload && systemctl enable --now kube-apiserver
$ systemctl status kube-apiserver
```

### ControllerManager

> 所有 Master 节点的 kube-controller-manager service 配置一样

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
  --address=127.0.0.1 \
  --root-ca-file=/etc/kubernetes/pki/ca.pem \
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \
  --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
  --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \
  --leader-elect=true \
  --use-service-account-credentials=true \
  --node-monitor-grace-period=40s \
  --node-monitor-period=5s \
  --pod-eviction-timeout=2m0s \
  --controllers=*,bootstrapsigner,tokencleaner \
  --allocate-node-cidrs=true \
  --cluster-cidr=172.10.0.0/12 \
  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem \
  --node-cidr-mask-size=24
      
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

所有 Master 节点开启服务并检查状态

```bash
$ systemctl daemon-reload && systemctl enable --now kube-controller-manager
$ systemctl status kube-controller-manager
```

### Scheduler

> 所有 Master 节点的 kube-scheduler service 配置一样

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
  --address=127.0.0.1 \
  --leader-elect=true \
  --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

所有 Master 节点开启服务并检查状态

```bash
$ systemctl daemon-reload && systemctl enable --now kube-scheduler
$ systemctl status kube-scheduler
```

##  TLS Bootstrapping

> 注：如需修改文件中的 token-id 和 token-secret，需要保持一致

在配置文件中的 token 定义中，需要注意：

- token 的 name 必须是 `bootstrap-token-<token-id>` 的格式
- token 的 type 必须是 `bootstrap.kubernetes.io/token`
- token 的 token-id 和 token-secret 分别是6位和16位数字和字母的组合
- `auth-extra-groups` 定义了 token 代表的用户所属的额外的 group，而默认 group 名为 `system:bootstrappers`
- 这种类型 token 代表的用户名为 `system:bootstrap:<token-id>`，在本文中就是 `system:bootstrap:abcdef`

1）Master01 操作即可，生成随便字符串

```bash
$ cat /dev/urandom | head -n 22 | md5sum | head -c 22
dd4414.74c103b52e297e6c
```

```bash
$ cd /root/k8s-install/bootstrap
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.120:8443 \
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
```


2）查看集群状态，正常即可继续操作
```bash
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

## Node

> Master01 节点将证书复制至其他节点

```bash
$ cd /etc/kubernetes/
$ for NODE in k8s-master02 k8s-master03 k8s-node01 k8s-node02; do
     ssh $NODE mkdir -p /etc/kubernetes/pki /etc/etcd/ssl /etc/etcd/ssl
     for FILE in etcd-ca.pem etcd.pem etcd-key.pem; do
       scp /etc/etcd/ssl/$FILE $NODE:/etc/etcd/ssl/
     done
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
CPUAccounting=true
MemoryAccounting=true
ExecStart=/usr/local/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
```

2）所有节点创建 Kubelet 配置文件

- Runtime 为 Docker，按以下内容创建 Kubelet 配置文件

```bash
$ vim /etc/systemd/system/kubelet.service.d/10-kubelet.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_CONFIG_ARGS=--config=/etc/kubernetes/kubelet-conf.yml --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node='' "
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_SYSTEM_ARGS $KUBELET_EXTRA_ARGS
```

- Runtime 为 Containerd，按以下内容创建 Kubelet 配置文件

```bash
$ vim /etc/systemd/system/kubelet.service.d/10-kubelet.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock --cgroup-driver=systemd"
Environment="KUBELET_CONFIG_ARGS=--config=/etc/kubernetes/kubelet-conf.yml"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node='' "
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_SYSTEM_ARGS $KUBELET_EXTRA_ARGS
```

```bash
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

启动所有节点 kubelet 服务

```bash
$ systemctl daemon-reload && systemctl enable --now kubelet
$ systemctl status kubelet
```

3）查看系统日志和集群状态

```bash
# 此时系统日志显示如下信息都是正常的
$ tail -f /var/log/messages
...
Unable to update cni config: no networks found in /etc/cni/net.d

$ kubectl get node
NAME           STATUS     ROLES    AGE   VERSION
k8s-master01   NotReady   <none>   38s   v1.23.9
k8s-master02   NotReady   <none>   38s   v1.23.9
k8s-master03   NotReady   <none>   38s   v1.23.9
k8s-node01     NotReady   <none>   39s   v1.23.9
k8s-node02     NotReady   <none>   38s   v1.23.9
```

### Kube-Proxy

> 仅在 Master01 执行

1）通过 Master01 获取参数值并生成 `kubeconfig` 文件

```bash
$ cd /root/k8s-install
$ kubectl -n kube-system create serviceaccount kube-proxy
$ kubectl create clusterrolebinding system:kube-proxy \
  --clusterrole system:node-proxier \
  --serviceaccount kube-system:kube-proxy
  
$ SECRET=$(kubectl -n kube-system get sa/kube-proxy \
  --output=jsonpath='{.secrets[0].name}')
$ JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET \
  --output=jsonpath='{.data.token}' | base64 -d)

$ PKI_DIR=/etc/kubernetes/pki
$ K8S_DIR=/etc/kubernetes
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem \
  --embed-certs=true \
  --server=https://188.188.4.120:8443 \
  --kubeconfig=${K8S_DIR}/kube-proxy.kubeconfig

$ kubectl config set-credentials kubernetes \
  --token=${JWT_TOKEN} \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

$ kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=kubernetes \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

$ kubectl config use-context kubernetes \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
```

2）将 kuberconfig 发至其他节点

```bash
$ for NODE in k8s-master02 k8s-master03; do
     scp /etc/kubernetes/kube-proxy.kubeconfig  $NODE:/etc/kubernetes/kube-proxy.kubeconfig
 done

$ for NODE in k8s-node01 k8s-node02; do
     scp /etc/kubernetes/kube-proxy.kubeconfig $NODE:/etc/kubernetes/kube-proxy.kubeconfig
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
```

```bash
$ vim /etc/kubernetes/kube-proxy.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.10.0.0/12 
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

所有节点启动 kube-proxy

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
$ curl https://projectcalico.docs.tigera.io/archive/v3.22/manifests/calico.yaml -o calico.yaml
$ curl https://projectcalico.docs.tigera.io/archive/v3.22/manifests/calico-typha.yaml -o calico.yaml
$ curl https://projectcalico.docs.tigera.io/archive/v3.22/manifests/calico-etcd.yaml -o calico.yaml
```

2）根据节点及需求选择版本安装

- **官方推荐版本**
  
```bash
$ curl https://projectcalico.docs.tigera.io/archive/v3.22/manifests/calico-typha.yaml -o calico.yaml
$ POD_SUBNET="172.10.0.0/12" 
$ sed -i 's@# - name: CALICO_IPV4POOL_CIDR@- name: CALICO_IPV4POOL_CIDR@g; s@#   value: "192.168.0.0/16"@  value: '"${POD_SUBNET}"'@g' calico.yaml
$ grep "IPV4POOL_CIDR" calico.yaml -A 1
            - name: CALICO_IPV4POOL_CIDR
              value: "172.10.0.0/12"

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
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/calico-typha created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
poddisruptionbudget.policy/calico-kube-controllers created

$ kubectl get po -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-dfbc9df67-fk7bj   1/1     Running   0          53s
calico-node-9gg8s                         1/1     Running   0          53s
calico-node-fcmls                         1/1     Running   0          53s
calico-node-qctzg                         1/1     Running   0          53s
calico-node-rdd8s                         1/1     Running   0          53s
calico-node-rjf8n                         1/1     Running   0          53s
calico-typha-849f97c74c-g8q6m             1/1     Running   0          53s 

$ crictl pods
POD ID              CREATED             STATE               NAME                                       NAMESPACE           ATTEMPT             RUNTIME
8028bb4dd5d6b       3 minutes ago       Ready               calico-kube-controllers-749f6598d9-prx25   kube-system         0                   (default)
b10b1e110ebb0       3 minutes ago       Ready               calico-node-mpx6m                          kube-system         0                   (default)

$ crictl images
IMAGE                                                       TAG                 IMAGE ID            SIZE
docker.io/calico/cni                                        v3.22.4             5f991a31345f7       77.9MB
docker.io/calico/kube-controllers                           v3.22.4             e60799840da95       52.3MB
docker.io/calico/node                                       v3.22.4             3a6768785333d       71.4MB
docker.io/calico/pod2daemon-flexvol                         v3.22.4             e2021682c0d9d       8.66MB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause   3.6                 6270bb605e12e       302kB
```

- **with etcd datastore**
  
  - 编辑 `kind: ConfigMap` 的 etcd_endpoints，添加3个 etcd 的 url 地址
  - 编辑 `kind: Secret` 中的内容，k8s 定义的证书 base64 的加密数据(etcd_ca、etcd_cert、etcd_key)，把此三个证书数据经过 base64 方式加密后，添加到 calico.yaml 中
  - 添加上述 etcd 证书的路径进行挂载
  - 编辑 `name: CALICO_IPV4POOL_CIDR` 添加 Pod IP 段(值与 `kube-contrller-manager` 的 systemd 配置文件 `--cluster-cidr` 或 `kubelet-conf.yaml` 的 `podCIDR` 一致 )

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220817100016.png)

```bash
$ curl https://projectcalico.docs.tigera.io/archive/v3.22/manifests/calico-etcd.yaml -o calico.yaml

$ sed -i 's#etcd_endpoints: "http://<ETCD_IP>:<ETCD_PORT>"#etcd_endpoints: "https://188.188.4.121:2379,https://188.188.4.122:2379,https://188.188.4.123:2379"#g' calico-etcd.yaml
$ ETCD_CA=`cat /etc/kubernetes/pki/etcd/etcd-ca.pem | base64 | tr -d '\n'`
$ ETCD_CERT=`cat /etc/kubernetes/pki/etcd/etcd.pem | base64 | tr -d '\n'`
$ ETCD_KEY=`cat /etc/kubernetes/pki/etcd/etcd-key.pem | base64 | tr -d '\n'`
$ sed -i "s@# etcd-key: null@etcd-key: ${ETCD_KEY}@g; s@# etcd-cert: null@etcd-cert: ${ETCD_CERT}@g; s@# etcd-ca: null@etcd-ca: ${ETCD_CA}@g" calico-etcd.yaml
$ sed -i 's#etcd_ca: ""#etcd_ca: "/calico-secrets/etcd-ca"#g; s#etcd_cert: ""#etcd_cert: "/calico-secrets/etcd-cert"#g; s#etcd_key: "" #etcd_key: "/calico-secrets/etcd-key" #g' calico-etcd.yaml
$ POD_SUBNET="172.10.0.0/12"
$ sed -i 's@# - name: CALICO_IPV4POOL_CIDR@- name: CALICO_IPV4POOL_CIDR@g; s@#   value: "192.168.0.0/16"@  value: '"${POD_SUBNET}"'@g' calico-etcd.yaml

$ kubectl apply -f calico.yaml
$ kubectl get po -n kube-system
```
注：如果容器状态异常可以使用 kubectl describe 或者 kubectl logs 查看容器的日志

## CoreDNS

> 安装前请参考 [CoreDNS GitHub](https://github.com/coredns/deployment/blob/master/kubernetes/CoreDNS-k8s_version.md) 版本介绍和 [yaml 示例文件](https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed)/[1.86 yaml示例文件](https://github.com/coredns/deployment/commit/8e13ce82cc607c53551dfe3ab40aef788a69c1ac)

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
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.2/components.yaml
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
        image: registry.cn-hongkong.aliyuncs.com/yuikuen/metrics-server:v0.5.2

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

| Kubernetes version | 1.20 | 1.21 | 1.22 | 1.23 |
| ------------------ | ---- | ---- | ---- | ---- |
| Compatibility      | ?    | ?    | ?    | ✓    |

- `✓` Fully supported version range.
- `?` Due to breaking changes between Kubernetes API versions, some features might not work correctly in the Dashboard.

1）下载兼容的版本文件

```bash
# 官方推荐版本
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml

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