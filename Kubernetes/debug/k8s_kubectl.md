# Kubectl 配置多集群管理

> 企业内部有测试、正式环境等多个集群，为了更高效、方便的访问管理集群，利用集群外主机通过 Kubectl 来管理

kubectl 版本和集群版本之间的差异必须在一个小版本号内（例如：v1.24 版本的客户端能与 v1.23、 v1.24 和 v1.25 版本的控制面通信）

## 安装 Kubectl

安装方法可以参考 [官方网站](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

- Curl 方式安装

1）下载最新版本

```bash
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

如需下载某个指定的版本，请用指定版本号替换该命令的这一部分：

```bash
$ curl -LO https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl
```

2）安装 Kubectl

```bash
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

3）测试查看版本

```bash
$ kubectl version --client
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.0", GitCommit:"4ce5a8954017644c5420bae81d72b09b735c21f0", GitTreeState:"clean", BuildDate:"2022-05-03T13:46:05Z", GoVersion:"go1.18.1", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.4
```

4）未配置文件前测试，提示无法找到集群

```bash
$ kubectl cluster-info
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
The connection to the server localhost:8080 was refused - did you specify the right host or port?

$ kubectl cluster-info dump
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

5）为当前客户端配置一个集群的访问凭证 `~/.kube/config`

为了让 kubectl 能发现并访问 k8s 集群，需要一个 `kubeconfig` 文件。通常在创建集群时，均自动生成，配置信息存放于 `~/.kube/config` 中

```bash
# 从集群中复制config文件此客户端
$ scp -r ~/.kube root@client:/root/
$ ls -a
.  ..  cache  config
```

6）再次验证，通过获取集群状态，检查是否正确配置了 kubectl

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://188.188.4.110:8443
CoreDNS is running at https://188.188.4.110:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get node
NAME      STATUS   ROLES    AGE   VERSION
dev-m01   Ready    <none>   3d    v1.23.9
dev-m02   Ready    <none>   3d    v1.23.9
dev-m03   Ready    <none>   3d    v1.23.9
dev-n01   Ready    <none>   3d    v1.23.9
dev-n02   Ready    <none>   3d    v1.23.9
```
如返回一个 URL，则 Client 的 kubectl 成功访问集群

## 配置多集群

> 具体场景：现需要使用专用的客户端主机，通过 `kubectl` 管理多个 K8s 集群

操作过程：将各集群的 `kubectl config` 文件中的证书内容转换，通过命令创建 config 文件，通过上下文切换使用不同集群

1）提前将各集群文件拷贝至客户端

```bash
# 客户端提前创建目录
$ mkdir -p ~/.kube

# prod-m01
$ scp -r ~/.kube/config root@client:/tmp/config1

# dev-m01
$ scp -r ~./kube/config root@client:/tmp/config2
```

2）修改配置文件，以区分集群

```bash
# prod-m01原文示例
$ cat config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    # 加密密钥
    server: https://188.188.4.120:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: 
    # 加密密钥


# dev-m01原文示例
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    # 加密密钥
    server: https://188.188.4.110:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: 
    # 加密密钥
    client-key-data: 
    # 加密密钥
```

只需要修改 `contexts` 部分内容即可

```bash
$ cat config1
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    server: https://188.188.4.120:8443
  # 修改name
  name: cluster1
contexts:
- context:
    # 对应上文cluster
    cluster: cluster1
    user: prod-admin
  name: context-cluster1
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
# 修改name
- name: prod-admin
  user:
    client-certificate-data: 

$ cat config2
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    server: https://188.188.4.110:8443
  # 修改name
  name: cluster2
contexts:
- context:
    # 对应上文cluster
    cluster: cluster2
    user: dev-admin
  name: context-cluster2
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
# 修改name
- name: dev-admin
  user:
    client-certificate-data: 
```

3）写入配置文件

```bash
$ KUBECONFIG=config1:config2 kubectl config view --flatten > $HOME/.kube/config

# 注意：因是新创建的，所以使用 > ~/.kube/config 将输出结果覆盖到config中，后续追加应改为 >> ~/.kube/config
$ KUBECONFIG=config1:config2 kubectl config view --flatten >> $HOME/.kube/config
```

4）测试配置

> 查看集群信息，* 表示当前的工作环境

```bash
[root@mw-02 .kube]#kubectl config get-contexts
CURRENT   NAME               CLUSTER    AUTHINFO     NAMESPACE
          context-cluster1   cluster1   prod-admin   
*         context-cluster2   cluster2   dev-admin    

[root@mw-02 .kube]# kubectl config current-context
context-cluster2

[root@mw-02 .kube]# kubectl get node
NAME      STATUS   ROLES    AGE    VERSION
dev-m01   Ready    <none>   3d1h   v1.23.9
dev-m02   Ready    <none>   3d1h   v1.23.9
dev-m03   Ready    <none>   3d1h   v1.23.9
dev-n01   Ready    <none>   3d1h   v1.23.9
dev-n02   Ready    <none>   3d1h   v1.23.9

[root@mw-02 .kube]# kubectl config use-context context-cluster1
Switched to context "context-cluster1".

[root@mw-02 .kube]# kubectl get node
NAME       STATUS   ROLES    AGE     VERSION
prod-m01   Ready    <none>   2d22h   v1.24.4
prod-m02   Ready    <none>   2d22h   v1.24.4
prod-m03   Ready    <none>   2d22h   v1.24.4
prod-n01   Ready    <none>   2d22h   v1.24.4
prod-n02   Ready    <none>   2d22h   v1.24.4

[root@mw-02 .kube]# kubectl get po -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS        AGE
kube-system            calico-kube-controllers-6754f569c-x48qf      1/1     Running   3 (2d21h ago)   2d22h
kube-system            calico-node-5bhs8                            1/1     Running   2 (2d21h ago)   2d22h
kube-system            calico-node-bz8p9                            1/1     Running   2 (2d21h ago)   2d22h
kube-system            calico-node-dhkhk                            1/1     Running   2 (2d21h ago)   2d22h
kube-system            calico-node-gl92w                            1/1     Running   2 (2d21h ago)   2d22h
kube-system            calico-node-mzlmw                            1/1     Running   2 (2d21h ago)   2d22h
kube-system            calico-typha-bb4f6c8b5-nnlkn                 1/1     Running   2 (2d21h ago)   2d22h
kube-system            coredns-754ffb777-dpzww                      1/1     Running   2 (2d21h ago)   2d22h
kube-system            metrics-server-794977b85f-6p65k              1/1     Running   2 (2d21h ago)   2d21h
kubernetes-dashboard   dashboard-metrics-scraper-55c6b4b959-qfc7h   1/1     Running   2 (2d21h ago)   2d21h
kubernetes-dashboard   kubernetes-dashboard-59d7b7b47b-8szv6        1/1     Running   4 (2d21h ago)   2d21h
```

**附录**

```bash
current-context 显示 current_context
delete-cluster  删除 kubeconfig 文件中指定的集群
delete-context  删除 kubeconfig 文件中指定的 context
get-clusters    显示 kubeconfig 文件中定义的集群
get-contexts    描述一个或多个 contexts
rename-context  Renames a context from the kubeconfig file.
set             设置 kubeconfig 文件中的一个单个值
set-cluster     设置 kubeconfig 文件中的一个集群条目
set-context     设置 kubeconfig 文件中的一个 context 条目
set-credentials 设置 kubeconfig 文件中的一个用户条目
unset           取消设置 kubeconfig 文件中的一个单个值
use-context     设置 kubeconfig 文件中的当前上下文
view            显示合并的 kubeconfig 配置或一个指定的 kubeconfig 文件
```

