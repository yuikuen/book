# K8s Install-1.20.x(Kubeadm)

> Kubeadm 部署高可用 K8s-1.20.x

## 版本环境

**程序版本**

安装时请注意版本控制，如安装失败可检查各程序版本之间是否有兼容问题

- System：CentOS7.9.2009 Minimal
- Runtime：docker-ce-19.03.*
- kubeadm：kubeadm-1.20* kubelet-1.20* kubectl-1.20*
- Calico：v3.20
- Metricd：v0.5.2
- Dashboard：v2.4.0

**安装规划**

- 配置环境：ESXI 6.7.0
  - 系统版本：CentOS-7.9-Minimal
  - Docker 版本：19.03.x
  - Pod 网段：172.168.0.0/12
  - Service 网段：10.96.0.0/12

注：宿主机网段、K8s Server 网段、Pod 网段不能重复

**集群网络规划**

| 主机名        | IP 地址       | 功能     |
| ------------- | ------------- | -------- |
| k8s-master-lb | 188.188.4.120 | VIP      |
| k8s-master01  | 188.188.4.121 | Master01 |
| k8s-master02  | 188.188.4.122 | Master02 |
| k8s-master03  | 188.188.4.123 | Master03 |
| k8s-node01    | 188.188.4.124 | Node01   |
| k8s-node02    | 188.188.4.125 | Node02   |

**常用工具安装**

```bash
$ yum install wget jq psmisc lrzsz vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git expect -y
```

## 环境配置

**所有节点配置 hosts 文件**

```bash
$ cat >> /etc/hosts << EOF
188.188.4.120 k8s-master-lb
188.188.4.121 k8s-master01
188.188.4.122 k8s-master02
188.188.4.123 k8s-master03
188.188.4.124 k8s-node01
188.188.4.125 k8s-node02
EOF
```

**所有节点配置公共源**

```bash
$ curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
$ yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

$ cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

$ sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

**所有节点关闭防火墙、selinux、dnsmasq、swap**

```bash
$ systemctl disable --now firewalld 
$ systemctl disable --now dnsmasq
$ systemctl disable --now NetworkManager

$ setenforce 0
$ sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/sysconfig/selinux
$ sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config

$ swapoff -a && sysctl -w vm.swappiness=0
$ sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

**所有节点安装 ntpdate 进行同步时间**

```bash
$ rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
$ yum install ntpdate -y
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
$ echo 'Asia/Shanghai' >/etc/timezone
$ ntpdate time2.aliyun.com
$ crontab -e
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com
```

**配置免密钥登录**

master01 节点生成证书，并发放到其他节点作免密登录，集群管理也在 master01 上操作；

```bash
$ ssh-keygen -t rsa -P "" -f /root/.ssh/id_rsa
$ for i in k8s-master01 k8s-master02 k8s-master03 k8s-node01 k8s-node02;do
expect -c "
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@$i
        expect {
                \"*yes/no*\" {send \"yes\r\"; exp_continue}
                \"*password*\" {send \"Password\r\"; exp_continue}
                \"*Password*\" {send \"Password\r\";}
        } "
done 
```

## 内核配置

**所有节点配置 Limit**

```bash
$ vim /etc/security/limits.conf
# 末尾添加如下内容
* soft nofile 65536
* hard nofile 131072
* soft nproc 65535
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
```

**所有节点升级系统并重启**

```bash
$ yum update -y --exclude=kernel* && reboot
```

```bash
# 通过下载 kernel image 的 rpm 包进行安装
$ rpm -import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
$ rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm

# ELRepo中有两个内核选项，kernel-lt(长期支持版本)，kernel-ml(主线最新版本)，采用长期支持版本(kernel-lt)，更稳定一些
$ yum -y install kernel-lt --enablerepo=elrepo-kernel

# 修改内核顺序
$ grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg
$ grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)" 

# 使用下面命令看看确认下是否启动默认内核指向上面安装的内核
$ grubby --default-kernel && reboot now
```

**所有节点安装 ipvsadm**

```bash
$ yum install ipvsadm ipset sysstat conntrack libseccomp -y
$ vim /etc/modules-load.d/ipvs.conf
# 加入以下内容
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

**所有节点开启 k8s 集群必须的内核参数**

```bash
$ cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
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

# 所有节点配置完内核后，重启服务器，保证重启后内核依旧加载
$ sysctl --system && init 6
$ lsmod | grep --color=auto -e ip_vs -e nf_conntrack
```

## 基本组件

**主要安装的是集群中用到的各种组件，比如 Docker-ce、Kubernetes 各组件等**

```bash
# 所有节点安装
$ yum install docker-ce-19.03.* docker-cli-19.03.* -y

# 由于新版kubelet建议使用systemd，所以可以把docker的CgroupDriver改成systemd
$ mkdir /etc/docker
$ cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

$ systemctl daemon-reload && systemctl enable --now docker
```

**所有节点安装最新版本 kubeadm**

```bash
# 查看k8s版本组件
$ yum list kubeadm.x86_64 --showduplicates | sort -r
$ yum install kubeadm-1.20* kubelet-1.20* kubectl-1.20* -y

# 默认配置的pause镜像使用gcr.io仓库，国内可能无法访问，所以这里配置Kubelet使用阿里云的pause镜像
$ cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2"
EOF

$ systemctl daemon-reload && systemctl enable --now kubelet
```

## 高可用组件

**所有 Master 节点通过 yum 安装 HAProxy 和 KeepAlived**

```bash
$ yum install keepalived haproxy -y
```

- 所有 Master 节点配置 HAProxy

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

frontend monitor-in
  bind *:33305
  mode http
  option httplog
  monitor-uri /monitor

frontend 4dev-m
  bind 0.0.0.0:16443
  bind 127.0.0.1:16443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend 4dev-m

backend 4dev-m
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-master01	188.188.4.121:6443  check
  server k8s-master02	188.188.4.122:6443  check
  server k8s-master03	188.188.4.123:6443  check
```

- 所有 Master 节点配置 KeepAlived (注意每个注意每个节点的 IP 和网卡 interface 参数)

```bash
# master01
$ vim /etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
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
    interface ens192
    mcast_src_ip 188.188.4.121
    virtual_router_id 51
    priority 101
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
    }
}
```

```bash
# master02
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
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
    interface ens192
    mcast_src_ip 188.188.4.122
    virtual_router_id 51
    priority 100
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
    }
}
```

```bash
# master03
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
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
    interface ens192
    mcast_src_ip 188.188.4.123
    virtual_router_id 51
    priority 100
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
    }
}
```

**所有 master 节点配置 KeepAlived 健康检查文件**

```bash
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
```

**启动 haproxy 和 keepalived**

```bash
$ systemctl daemon-reload
$ systemctl enable --now haproxy keepalived
```

## 集群初始化

**Master01 节点创建 kubeadm-config.yaml**

<font color=red>注：如不是高可用集群，无需配置 `高可用组件`，并且需要将下列的 vip-ip:16443 改为master01 的地址，而 16443 改为 apiserver 的默认端口 6443</font>

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: 7t2weq.bjbawausm0jaxury
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 188.188.4.121
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - 188.188.4.120
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 188.188.4.120:16443
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.20.13
networking:
  dnsDomain: cluster.local
  podSubnet: 172.168.0.0/12
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

**更新 kubeadm 文件**

```bash
$ kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
$ kubeadm config images pull --config /root/new.yaml
# 所有节点设置开机自启动（如失败无法理会，初始化成功后即可启动）
$ systemctl enable --now kubelet
```

*注*：将更新后的文件 `new.yaml` 文件复制到其他 Master 节点，便于所有 Master 节点提前下载镜像

**Master01 节点初始化**

初始化后会有 `/etc/kubernetes` 目录下生成对应的证书和配置文件，之后其他 Master 节点加入 Master01 即可

```bash
$ kubeadm init --config /root/new.yaml --upload-certs
```

- 初始化失败：需要重置再次初始化

```bash
$ kubeadm reset -f ; ipvsadm --clear  ; rm -rf ~/.kube
```

- 初始化成功：会产生 Token 值，用于其他节点加入时使用

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 188.188.4.120:16443 --token 7t2weq.bjbawausm0jaxury \
    --discovery-token-ca-cert-hash sha256:ef7f6e2707c4f3eed43252ebe58d2e13020412f6f436e497a62e476efb721fca \
    --control-plane --certificate-key 5068c01712f2da51285969a5f1b90383433e637ed5606a93be94065a2d09b7bd

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 188.188.4.120:16443 --token 7t2weq.bjbawausm0jaxury \
    --discovery-token-ca-cert-hash sha256:ef7f6e2707c4f3eed43252ebe58d2e13020412f6f436e497a62e476efb721fca
```

**Master01 节点配置环境变量，用于访问集群**

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ cat <<EOF >> /root/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF
$ source /root/.bashrc
```

采用初始化安装方式的，所有系统组件均以容器方式运行，并且在 kube-system 命令空间内

```bash
$ kubectl get node
NAME           STATUS     ROLES                  AGE     VERSION
k8s-master01   NotReady   control-plane,master   6m6s    v1.20.13
k8s-master02   NotReady   control-plane,master   3m13s   v1.20.13
k8s-master03   NotReady   control-plane,master   2m19s   v1.20.13
k8s-node01     NotReady   <none>                 51s     v1.20.13
k8s-node02     NotReady   <none>                 47s     v1.20.13

$ kubectl get po -A -owide
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
kube-system   coredns-54d67798b7-4f85f               0/1     Pending   0          6m12s   <none>          <none>         <none>           <none>
kube-system   coredns-54d67798b7-m94vj               0/1     Pending   0          6m12s   <none>          <none>         <none>           <none>
kube-system   etcd-k8s-master01                      1/1     Running   0          6m12s   188.188.4.121   k8s-master01   <none>           <none>
kube-system   etcd-k8s-master02                      1/1     Running   0          3m25s   188.188.4.122   k8s-master02   <none>           <none>
kube-system   etcd-k8s-master03                      1/1     Running   0          2m31s   188.188.4.123   k8s-master03   <none>           <none>
kube-system   kube-apiserver-k8s-master01            1/1     Running   0          6m12s   188.188.4.121   k8s-master01   <none>           <none>
kube-system   kube-apiserver-k8s-master02            1/1     Running   0          3m26s   188.188.4.122   k8s-master02   <none>           <none>
kube-system   kube-apiserver-k8s-master03            1/1     Running   0          77s     188.188.4.123   k8s-master03   <none>           <none>
kube-system   kube-controller-manager-k8s-master01   1/1     Running   1          6m11s   188.188.4.121   k8s-master01   <none>           <none>
kube-system   kube-controller-manager-k8s-master02   1/1     Running   0          3m25s   188.188.4.122   k8s-master02   <none>           <none>
kube-system   kube-controller-manager-k8s-master03   1/1     Running   0          99s     188.188.4.123   k8s-master03   <none>           <none>
kube-system   kube-proxy-5gp6j                       1/1     Running   0          64s     188.188.4.124   k8s-node01     <none>           <none>
kube-system   kube-proxy-bq8gg                       1/1     Running   0          6m13s   188.188.4.121   k8s-master01   <none>           <none>
kube-system   kube-proxy-mprpf                       1/1     Running   0          2m32s   188.188.4.123   k8s-master03   <none>           <none>
kube-system   kube-proxy-qkz6z                       1/1     Running   0          3m26s   188.188.4.122   k8s-master02   <none>           <none>
kube-system   kube-proxy-tsvk9                       1/1     Running   0          60s     188.188.4.125   k8s-node02     <none>           <none>
kube-system   kube-scheduler-k8s-master01            1/1     Running   1          6m12s   188.188.4.121   k8s-master01   <none>           <none>
kube-system   kube-scheduler-k8s-master02            1/1     Running   0          3m25s   188.188.4.122   k8s-master02   <none>           <none>
kube-system   kube-scheduler-k8s-master03            1/1     Running   0          90s     188.188.4.123   k8s-master03   <none>           <none>
```

## Calico 安装

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211124111602256.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211124112502719.png)

- [Calico 安装要求](https://docs.projectcalico.org/getting-started/kubernetes/requirements)
- [Calico Github](https://github.com/projectcalico/calico)
- [Calico-Etcd Yaml 参考文件](https://docs.projectcalico.org/archive/v3.20/manifests/calico-etcd.yaml)

Master01 节点执行即可

```bash
# 最新版本
$ curl https://docs.projectcalico.org/manifests/calico-etcd.yaml -o calico-etcd.yaml
# 可选择性版本下载
$ curl https://docs.projectcalico.org/archive/v3.20/manifests/calico-etcd.yaml -o calico-etcd.yaml

# 修改calico-etcd.yaml配置文件
$ sed -i 's#etcd_endpoints: "http://<ETCD_IP>:<ETCD_PORT>"#etcd_endpoints: "https://188.188.4.121:2379,https://188.188.4.122:2379,https://188.188.4.123:2379"#g' calico-etcd.yaml
$ ETCD_CA=`cat /etc/kubernetes/pki/etcd/ca.crt | base64 | tr -d '\n'`
$ ETCD_CERT=`cat /etc/kubernetes/pki/etcd/server.crt | base64 | tr -d '\n'`
$ ETCD_KEY=`cat /etc/kubernetes/pki/etcd/server.key | base64 | tr -d '\n'`
$ sed -i "s@# etcd-key: null@etcd-key: ${ETCD_KEY}@g; s@# etcd-cert: null@etcd-cert: ${ETCD_CERT}@g; s@# etcd-ca: null@etcd-ca: ${ETCD_CA}@g" calico-etcd.yaml
$ sed -i 's#etcd_ca: ""#etcd_ca: "/calico-secrets/etcd-ca"#g; s#etcd_cert: ""#etcd_cert: "/calico-secrets/etcd-cert"#g; s#etcd_key: "" #etcd_key: "/calico-secrets/etcd-key" #g' calico-etcd.yaml
$ POD_SUBNET=`cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr= | awk -F= '{print $NF}'`
# 修改默认配置的网段信息并去除注释（如替换过请自行改回）
$ sed -i 's@# - name: CALICO_IPV4POOL_CIDR@- name: CALICO_IPV4POOL_CIDR@g; s@#   value: "192.168.0.0/16"@  value: '"${POD_SUBNET}"'@g' calico-etcd.yaml
$ kubectl apply -f calico-etcd.yaml
```

安装后，重新获取查看状态

```bash
$ kubectl get node
NAME           STATUS   ROLES                  AGE    VERSION
k8s-master01   Ready    control-plane,master   2d3h   v1.20.13
k8s-node01     Ready    <none>                 2d3h   v1.20.13
k8s-node02     Ready    <none>                 2d3h   v1.20.13

$ kubectl get po -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-6d57448946-47795   1/1     Running   1          27h
kube-system   calico-node-bdhxc                          1/1     Running   1          27h
kube-system   calico-node-rdxpb                          1/1     Running   1          27h
kube-system   calico-node-s5qjc                          1/1     Running   2          27h
kube-system   coredns-54d67798b7-7kz57                   1/1     Running   0          28s
kube-system   coredns-54d67798b7-fwsrp                   1/1     Running   0          28s
kube-system   etcd-k8s-master01                          1/1     Running   2          2d3h
kube-system   kube-apiserver-k8s-master01                1/1     Running   2          44h
kube-system   kube-controller-manager-k8s-master01       1/1     Running   2          2d3h
kube-system   kube-proxy-8dvz2                           1/1     Running   2          2d3h
kube-system   kube-proxy-bq7tl                           1/1     Running   2          2d3h
kube-system   kube-proxy-r4rbb                           1/1     Running   2          2d3h
kube-system   kube-scheduler-k8s-master01                1/1     Running   2          2d3h
```

## Metrics 安装

在新版的 Kubernetes 中，系统资源的采集均使用 Metrics-server，可通过 Metrics 采集节点和 Pod 的内存、磁盘、CPU 和网络的使用率。

将 Master01 节点的 `front-proxy-ca.crt` 复制到其它 Node 节点

```bash
$ for node in k8s-master02 k8s-master03 k8s-node01 k8s-node02; do
     scp /etc/kubernetes/pki/front-proxy-ca.crt $node:/etc/kubernetes/pki/front-proxy-ca.crt
  done
```

[Metrics-Server Github 地址](https://github.com/kubernetes-sigs/metrics-server/)

**查看版本兼容是否合适**

| Metrics Server | Metrics API group/version | Supported Kubernetes version |
| -------------- | ------------------------- | ---------------------------- |
| 0.6.x          | `metrics.k8s.io/v1beta1`  | *1.19+                       |
| 0.5.x          | `metrics.k8s.io/v1beta1`  | *1.8+                        |
| 0.4.x          | `metrics.k8s.io/v1beta1`  | *1.8+                        |
| 0.3.x          | `metrics.k8s.io/v1beta1`  | 1.8-1.21                     |

```bash
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.2/components.yaml
```

提前下载镜像并修改镜像地址信息和添加证书认证，否则无法下载到镜像

[配置聚合层](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)，支持代理服务器和扩展 apiserver 之间的相互 TLS 身份验证

```yaml
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
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt # change to front-proxy-ca.crt for kubeadm
        - --requestheader-username-headers=X-Remote-User
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        image: registry.cn-hongkong.aliyuncs.com/yk-k8s/metrics-server:v0.5.2

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
```

安装后查看状态

```bash
$ kubectl get po -n kube-system | grep metrics
metrics-server-86bf587fcd-sfn5r            1/1     Running   0          2m57s

$ kubectl top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01   228m         3%     937Mi           7%        
k8s-master02   229m         3%     861Mi           7%        
k8s-master03   246m         4%     968Mi           8%        
k8s-node01     121m         3%     592Mi           4%        
k8s-node02     131m         3%     593Mi           4% 
```

## Dashboard

Dashboard 用于展示集群中的各类资源，同时也可以通过 Dashboard 实时查看 Pod 的日志和在容器中执行一些命令等

[Dashboard Github 地址 ](https://github.com/kubernetes/dashboard)

**首先创建管理员 **

```bash
$ cat admin.yaml
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

```bash
# 下载最新版本的Dashboard
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml
$ kubectl apply -f .
```

注意：使用 google-chrome 浏览器启动前，修改启动参数，解决无法访问 Dashboard  的问题

```bash
--test-type --ignore-certificate-errors
```

更改 Dashboard 的 svc 为 NodePort

```yaml
$ kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
ports:
- port: 443
  protocol: TCP
  targetPort: 8443
selector:
  k8s-app: kubernetes-dashboard
sessionAffinity: None
type: ClusterIP
```
将 ClusterIP 更改为 NodePort

```yaml
ports:
- port: 443
  protocol: TCP
  targetPort: 8443
  nodePort: 30000
selector:
  k8s-app: kubernetes-dashboard
sessionAffinity: None
type: NodePort
```

查看更新后的服务端口情况

```bash
$ kubectl get svc kubernetes-dashborad -n kubernetes-dashboard
```

根据显示的实例端口，通过任意已安装了 kube-proxy 的宿主机或 VIP 的 IP+端口即可访问

**查看 token 值**

```bash
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-fbjpl
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 77249734-ca1f-49aa-b29e-beaa5c90ae33

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkVXQTRNTjN4LUZQVVBmXzJ2UU5jNE5oeVc1LURkVGw1eGRvVTJIQ19jeGsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWZianBsIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3NzI0OTczNC1jYTFmLTQ5YWEtYjI5ZS1iZWFhNWM5MGFlMzMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.jBbVbgZXFjr2KRy_8_83ht6N4b2jZMgFpCFByhzxtJOJPA35BKJsAuKhgV-TZ0Uk2_jpE2RDNt4PCoOOCKFcuPGnsEumbNcmjwh0q2w-OTzcuohhkJRhi34haa4MafcbqUtsrHszPPw1o71TZ_Ex_ydQr7OSmljfxtkdYvtAT4RJ7g7ygNpPnMuPB7cpIjTHCwrwFcM9HAnJLkJqjlSehPZeg5R8Shs4lU2TTTLn4LfbVKuS6qCG8PHNW9iJ1IDQfjguIEd3WXiMUoO5DpjY_v6M-nZnRi-1cLGkMkWVvjSqqsnVZH-zdoMj6mp1G1pA9HdXIMsMSzGbMsMpTeiRnA
```

将 token 值输入到令牌后，单击登录即可访问 Dashboard

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207100817690.png)

## 缺失部分

**Kube-proxy 改为 ipvs 模式**

因在初始化集群时注释了 ipvs 配置，所以需要自行修改一下；

```bash
# master01节点操作
$ kubectl edit cm kube-proxy -n kube-system
# nu:44
mode: "ipvs"
```

更新 kube-proxy 的 Pod

```bash
$ kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system
```

验证 kube-proxy 模式

```bash
$ curl 127.0.0.1:10249/proxyMode
# 初始化后默认为iptables
ipvs
```

## 注意事项

**Token 过期**

注：kubeadm 安装的集群，证书有效期默认为一年。以下步骤是上述 init 命令产生的 Token 过期了才需要执行以下步骤，如未过期无需执行

> 通过kubeadm初始化之后，都会提供node加入的token，默认的token的有效期是24小时，一般操作是重新生成新token，再获取ca证书sha256编码hash值，最后把节点加入到集群中。

```bash
# 查看token时效
$ kubeadm token list
```

- Token 过期后生成新的 Token

```bash
$ kubeadm token create --print-join-command
```

*注意：这样生成的 Token 有效期为24小时，如不想过期，可加上 `--ttl=0` 参数*

- Master 需要生成 `--certificate-key`

```bash
$ kubeadm init phase upload-certs --upload-certs
```

- Token 未过期就直接执行 join 就行了

```bash
  kubeadm join 188.188.4.120:16443 --token 7t2weq.bjbawausm0jaxury \
    --discovery-token-ca-cert-hash sha256:ef7f6e2707c4f3eed43252ebe58d2e13020412f6f436e497a62e476efb721fca \
    --control-plane --certificate-key 5068c01712f2da51285969a5f1b90383433e637ed5606a93be94065a2d09b7bd
    
  kubeadm join 188.188.4.120:16443 --token 7t2weq.bjbawausm0jaxury \
    --discovery-token-ca-cert-hash sha256:ef7f6e2707c4f3eed43252ebe58d2e13020412f6f436e497a62e476efb721fca
```

**节点组件**

Kubeadm 与二进制的区别，Kubeadm 的 master 节点中的 kube-apiserver、kube-scheduler、kube-controller-manager、etcd 都是以容器方式运行的，可以通过 `kubectl get po -n kube-system` 查看。

- kubelet 的配置文件在 `/etc/sysconfig/kubelet` 和 `/var/lib/kubelet/config.yaml`，修改后需要重启 kubelet 进程；
- 其它组件的配置文件在 `/etc/kubernetes/manifests` 目录下，比如 kube-apiserver.yaml，该 yaml 文件更改后，kubelet 会自动刷新配置，也就会重启 pod。(不能再次创建文件)
- kube-proxy 的配置在 kube-system 命名空间下的 configmap 中，可以通过 `kubectl edit cm kube-proxy -n kube-system` 进行更改，更改完成后，需要通过 patch 重启 kube-proxy

```bash
$ kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system
```

**资源利用**

Node 节点上主要部署公司的一些业务应用，生产环境中不建议 Master 节点部署系统组件之外的其他 Pod，测试环境可以允许 Master 节点部署 Pod 以节省系统资源，可以通过以下方式打开：

```bash
# 查看污点并将Master节点污点去除
$ kubectl  describe node -l node-role.kubernetes.io/master=  | grep Taints
Taints:             node-role.kubernetes.io/master:NoSchedule
Taints:             node-role.kubernetes.io/master:NoSchedule
Taints:             node-role.kubernetes.io/master:NoSchedule

$ kubectl  taint node  -l node-role.kubernetes.io/master node-role.kubernetes.io/master:NoSchedule-
node/k8s-master01 untainted
node/k8s-master02 untainted
node/k8s-master03 untainted

$ kubectl  describe node -l node-role.kubernetes.io/master=  | grep Taints
Taints:             <none>
Taints:             <none>
Taints:             <none>
```

**参考文献**

[更多学习内容请参考-K8s全栈架构师(基于世界五百强生产经验研发)](https://ke.qq.com/course/2738602)
