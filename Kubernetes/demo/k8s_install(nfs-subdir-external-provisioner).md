# K8s Install(NFS-Subdir-External-Provisioner)

> Kubernetes 创建 NFS 动态存储

## 环境说明

- 硬件系统：ESXI 6.7.0 Update 1
- 操作系统：CentOS 7.9
- Docker 版本：20.10.6
- Kubernetes 版本：1.21.x+HA高可用
- NFS Subdir External Provisioner 版本：v4.0.10

[NFS Subdir External Provisioner Github 地址]:https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

**NFS-Subdir-External-Provisioner 简介**

存储组件 `NFS subdir external provisioner` 是一个存储资源自动调配器，它可用将现有的 NFS 服务器通过持久卷声明来支持 Kubernetes 持久卷的动态分配。自动新建的文件夹将被命名为 `${namespace}-${pvcName}-${pvName}` ，由三个资源名称拼合而成

> 此组件是对 nfs-client-provisioner 的扩展，nfs-client-provisioner 已经不提供更新，且 nfs-client-provisioner 的 Github 仓库已经迁移到 [NFS-Subdir-External-Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) 的仓库

## 创建 NFS Server

先创建 NFS Server 端才能够正常使用 NFS 文件系统，参考文档：CentOS7部署_NFS服务器

**配置环境**

```bash
# 停止并禁用防火墙
$ systemctl disable --now firewalld

# 关闭并禁用 SELinux
$ setenforce 0
$ sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

**安装配置**

```bash
$ yum -y install rpcbind nfs-utils

# 创建目录并更改权限
$ mkdir -p /nfs
$ chown -R nfsnobody:nfsnobody /nfs
```

```bash
# 编辑exports
$ vim /etc/exports

# 输入以下内容(格式：FS共享的目录 NFS客户端地址1(参数1,参数2,...) 客户端地址2(参数1,参数2,...))
/nfs 188.188.4.0/24(rw,async,no_root_squash)

# 重启 NFS Server 并设置开机就启动
$ systemctl enable --now nfs && systemctl restart nfs rpcbind
```

如果设置为 `/nfs *(rw,async,no_root_squash)` 则对所以的 `IP` 都有效

- 常用选项：

  - ro：客户端挂载后，其权限为只读，默认选项；
  - rw:读写权限；
  - sync：同时将数据写入到内存与硬盘中；
  - async：异步，优先将数据保存到内存，然后再写入硬盘；
  - Secure：要求请求源的端口小于1024

- 用户映射：

  - root_squash:当NFS客户端使用root用户访问时，映射到NFS服务器的匿名用户；
  - no_root_squash:当NFS客户端使用root用户访问时，映射到NFS服务器的root用户；
  - all_squash:全部用户都映射为服务器端的匿名用户；
  - anonuid=UID：将客户端登录用户映射为此处指定的用户uid；
  - anongid=GID：将客户端登录用户映射为此处指定的用户gid

## 挂载 NFS Client

```bash
$ yum -y install rpcbind nfs-utils
# 创建挂载的文件夹，并挂载 nfs
$ mkdir -p /nfs/storage

# 不要把挂载项写到/etc/fstab文件中，因为开机时先挂载本机磁盘再启动网络，而NFS是需要网络启动后才能挂载的，所以我们把挂载命令写入到/etc/rc.d/rc.local文件中即可
# 因为在centos7中/etc/rc.d/rc.local的权限被降低了，所以需要赋予其可执行权
$ chmod +x /etc/rc.d/rc.local
# 编写自动启动脚本
$ vim /usr/local/sbin/nfsboot.sh
#! /bin/bash
## This is nfs自启动 shell script.
date
mount -t nfs 188.188.3.110:/nfs/storage /nfs/storage
echo " nfs自启动 success！！"
$ chmod +x /usr/local/sbin/nfsboot.sh
# 打开/etc/rc.d/rc.local文件，在末尾增加如下内容
$ vim /etc/rc.d/rc.local
/usr/local/sbin/nfsboot.sh

$ systemctl enable --now nfs rpcbind
```
**下载示例文件**

```bash
$ git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
$ cd nfs-subdir-external-provisioner/deploy
```

## 创建 ServiceAccount

创建一个拥有一定权限的 ServiceAccount 与后面要部署的 NFS Subdir Externa Provisioner 组件绑定

> 默认 Namespace 为 default，根据实际进行修改自身的 Namespace 

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

```bash
$ kubectl create -f rbac.yaml
```

## 创建 NFS-Subdir-External-Provisioner

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate                # 设置升级策略为删除再创建(默认为滚动更新)
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
        # image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          image: registry.cn-hongkong.aliyuncs.com/yk-k8s/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME     # Provisioner的名称,以后设置的storageclass要和这个保持一致
            # value: k8s-sigs.io/nfs-subdir-external-provisioner
              value: nfs-client              
            - name: NFS_SERVER           # NFS服务器地址,需和valumes参数中配置的保持一致
            # value: 10.3.243.101
              value: 188.188.4.161
            - name: NFS_PATH             # NFS服务器数据存储目录,需和valumes参数中配置的保持一致
            # value: /ifs/kubernetes
              value: /nfs
      volumes:
        - name: nfs-client-root
          nfs:
          # server: 10.3.243.101
          # path: /ifs/kubernetes
            server: 188.188.4.161         # NFS服务器地址
            path: /nfs                    # NFS服务器数据存储目录
```

> 由于官方镜像存储在 gcr.io 仓库中，国内无法拉取，请自行将其拉下并存储在阿里云仓库中。

```bash
$ kubectl create -f deployment.yaml
```

## 创建 StorageClass

创建 PVC 时经常需要指定 storageClassName 名称，这个参数配置的就是一个 StorageClass 资源名称，PVC 通过指定该参数来选择使用哪个 StorageClass，并与其关联的 Provisioner 组件来动态创建 PV 资源。所以，这里我们需要提前创建一个 Storagelcass 资源

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"   # 是否设置为默认的storageclass
# provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
provisioner: nfs-client                                    # 动态卷分配者名称，必须和上面创建的"provisioner"变量中设置的Name一致
parameters:
# archiveOnDelete: "false"
  archiveOnDelete: "true"                                  # 设置为"false"时删除PVC不会保留数据,"true"则保留数据
mountOptions: 
  - hard                                                   # 指定为硬挂载方式
  - nfsvers=4                                              # 指定NFS版本,这个需要根据NFS Server版本号设置
```

```bash
$ kubectl create -f class.yaml
```

## 创建 PVC 测试

创建一个用于测试的 PVC 资源部署到 Kubernetes 中，这样可以测试 NFS-Subdir-External-Provisioner 是否能够自动创建 PV 与该 PVC 进行绑定

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: managed-nfs-storage  # 需要与上面创建的storageclass的名称一致
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

```bash
$ kubectl create -f test-claim.yaml
```

**观察是否自动创建 PV 并与 PVC 绑定**

等待创建完成后观察 NFS-Subdir-External-Provisioner  是否会自动创建 PV 与该 PVC 进行绑定，可以执行下面命令:

```bash
$ kubectl get pvc test-claim -o yaml | grep phase
  phase: Bound
```

如果显示 `phase` 为 `Bound`，则说明已经创建 PV 且与 PVC 进行了绑定

如创建后，状态一直为`Pending`，可参考`Kubernetes-部署 NFS Provisioner 存储卷报错.md`进行处理，主要原因为：

- `chmod 755 -R /nfs/data`，按实际分配目录权限；
- 通过修改 kube-apiserver 的配置文件，增加参数 `--feature-gates=RemoveSelfLink=false`，开启默认禁用的参数 selfLink

## 创建 Pod 测试

创建一个测试的 Pod 资源部署到 Kubernetes 中，这样就可以测试上面创建的 PVC 是否能够正常使用

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: gcr.io/google_containers/busybox:1.24
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"    # 创建一个名称为"SUCCESS"的文件
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```

```bash
$ kubectl create -f test-pod.yaml
```

**进入 NFS Server 服务器验证是否存在测试文件**

进入 NFS Server 服务器的 NFS 挂载目录，检查在 Pod 中创建的文件 SUCCESS 是否存在：

```bash
$ cd /nfs && ls -l | grep test-claim
drwxrwxrwx. 2 root root 21 May 31 14:04 default-test-claim-pvc-4e1cc487-aa6e-460b-9c96-0e3c834fee67

$ cd default-test-claim-pvc-4e1cc487-aa6e-460b-9c96-0e3c834fee67 && ls -l
total 0
-rw-r--r--. 1 root root 0 May 31 14:04 SUCCESS
```

可以看到已经生成 `SUCCESS` 该文件，并且可知通过 NFS-Subdir-External-Provisioner 创建的目录命名方式为 `namespace名称-pvc名称-pv名称`，PV 名称是随机字符串，所以每次只要不删除 PVC，那么 Kubernetes 中的与存储绑定将不会丢失，要是删除 PVC 也就意味着删除了绑定的文件夹，下次就算重新创建相同名称的 PVC，生成的文件夹名称也不会一致，因为 PV 名是随机生成的字符串，而文件夹命名又跟 PV 有关,所以删除 PVC 需谨慎

## 删除资源测试

在测试组件是否能正常使用后，我们需要将上面的测试资源文件进行清理，可以执行下面命令:

```bash 
# 删除测试的 Pod 资源文件
$ kubectl delete -f test-pod.yaml 

# 删除测试的 PVC 资源文件
$ kubectl delete -f test-claim.yaml

# 删除后确认文件是否还存在
$ cd /nfs && ls -l
total 0
drwxrwxrwx. 2 root root 21 May 31 14:04 archived-default-test-claim-pvc-4e1cc487-aa6e-460b-9c96-0e3c834fee67
```
