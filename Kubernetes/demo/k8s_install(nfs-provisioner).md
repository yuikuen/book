# K8s Install(NFS-Provisioner)

> Kubernetes 部署 NFS-Provisioner 动态分配卷

## 环境说明

- 硬件系统：ESXI 6.7.0 Update 1
- 操作系统：CentOS 7.9
- Docker 版本：20.10.6
- Kubernetes 版本：1.21.x+HA高可用
- NFS Provisioner 版本：latest

**NFS Provisioner 简介**

NFS Provisioner 是一个自动配置卷程序，它使用现有的和已配置的 NFS 服务器来支持通过持久卷声明动态配置 Kubernetes 持久卷
持久卷被配置为：`namespace-{namespace}-namespace−{pvcName}-${pvName}`

[NFS Provisioner Github 地址](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-clien)

## 创建 NFS Server

参考文档：CentOS7部署_NFS服务器

1）下载示例文件

```bash
$ git clone https://github.com/kubernetes-retired/external-storage.git
$ cd external-storage/nfs-client/deploy/ 
```

2）创建 ServiceAccount

创建一个一定权限的 ServiceAccount 与后面要创建的 “NFS Provisioner” 绑定，赋予一定的权限

- 默认为 Namespace 为 default，根据实际情况进行修改；

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

3）部署 NFS Provisioner

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
    type: Recreate          # 设置升级策略为删除再创建(默认为滚动更新)
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
        # image: quay.io/external_storage/nfs-client-provisioner:latest
          image: registry.cn-hongkong.aliyuncs.com/yuikuen/nfs-client-provisioner::latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs    # 名称需要跟之后的storageclass保持一致
            - name: NFS_SERVER
            # value: 10.10.10.60       
              value: 188.188.4.161     # NFS服务器地址，与volumes保持一致           
            - name: NFS_PATH
            # value: /ifs/kubernetes   
              value: /nfs/data         # NFS服务器目录，与volumes保持一致
      volumes:
        - name: nfs-client-root
          nfs:
          # server: 10.10.10.60       
          # path: /ifs/kubernetes
            server: 188.188.4.161      # NFS服务器地址
            path: /nfs/data            # NFS服务器目录
```

```bash
$ kubectl create -f deployment.yaml
```

4）创建 NFS StorageClass

创建一个 StoageClass，声明 NFS 动态卷提供者名称为 “nfs-storage”

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: fuseim.pri/ifs # 动态卷分配者名称，必须和上面创建的"provisioner"变量中设置的Name一致
parameters:
# archiveOnDelete: "false"
  archiveOnDelete: "true"
```

```bash
$ kubectl create -f class.yaml
```

## 验证测试

1）创建一个测试用的 PVC 并观察是否自动创建是 PV 与其绑定

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  storageClassName: managed-nfs-storage   # 需要与上面创建的storageclass的名称一致
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

**查看 PVC 状态是否与 PV 绑定**

利用 Kubectl 命令获取 pvc 资源，查看 STATUS 状态是否为 “Bound”

```bash
$ kubectl get pvc
NAME       STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS
test-pvc   Bound    pvc-be0808c2-9957-11e9 1Mi        RWO            nfs-storage
```

如创建后，状态一直为`Pending`，可参考`Kubernetes-部署 NFS Provisioner 存储卷报错.md`进行处理，主要原因为：

- `chmod 755 -R /nfs/data`，按实际分配目录权限；
- 通过修改 kube-apiserver 的配置文件，增加参数 `--feature-gates=RemoveSelfLink=false`，开启默认禁用的参数 selfLink

2）创建一个测试用的 Pod，指定存储为上面创建的 PVC，然后创建一个文件在挂载的 PVC 目录中，然后进入 NFS 服务器下查看该文件是否存入其中

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
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```

进入 NFS Server 服务器的 NFS 挂载目录，查看是否存在 Pod 中创建的文件：

```bash
$ cd /nfs/data
$ ls -l

-rw-r--r-- 1 root root 0 Jun 28 12:44 SUCCESS
```
