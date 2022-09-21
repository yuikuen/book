# Helm Install(v3.5.4)

## 版本环境

- 硬件系统：ESXI 6.7.0 Update 1
- 操作系统：CentOS 7.9
- Docker 版本：20.10.6
- Kubernetes 版本：1.21.x+HA高可用
- Helm 版本：v3.5.4

## Helm 介绍

[Helm](https://helm.sh/) 是一款能够帮助你管理 Kubernetes 应用的程序，它可以让你创建自己的应用模板（chart），然后模板来创建配置很多可自定义参数，每次我们只需要设定很少或者不设置参数（使用默认参数）就可以将应用部署到 Kubernetes 中，后期就可以通过 Helm 来进行升级、回滚、删除等等操作的管理。

Helm 的 Charts 类似于应用商店，里面存有很多公司提供不同应用的模板，例如常部署的 Redis、Mysql、Nginx 等等，可以让我们很方便的采用别人的模板，然后进行一定的配置，就能在我们的 Kubernetes 集群中创建对应的应用。

Helm 还经常与 CI\CD 配置使用，在这个过程中用于维护应用程序的安装、升级、回滚等操作

## Helm 安装

1）访问 [Helm_Github 下载页面](https://github.com/helm/helm) 找到最新的客户端，里面有不同系统下的包，这里我们选择 Linux amd64，然后在 Linux 系统中使用 Wget 命令进行下载

```bash
# 下载Helm客户端
$ wget https://get.helm.sh/helm-v3.5.4-linux-amd64.tar.gz
```

2）接下来解压下载的包，然后将客户端放置到 /usr/local/bin/ 目录下：

```bash
# 解压 Helm
$ tar -zxvf helm-v3.5.4-linux-amd64.tar.gz
# 复制客户端执行文件到 bin 目录下，方便在系统下能执行 helm 命令
$ cp linux-amd64/helm /usr/local/bin/
```

> 注意：helm 客户端需要下载到安装了 kubectl 并且能执行能正常通过 kubectl 操作 kubernetes 的服务器上，否则 helm 将不可用

3）添加 [Chart Repo](https://artifacthub.io/)，在 Helm 中默认是不会添加 Chart 仓库，所以这里我们需要手动添加，下面是添加一些常用的 Charts 库：

```bash
$ helm repo add  elastic    https://helm.elastic.co
$ helm repo add  gitlab     https://charts.gitlab.io
$ helm repo add  harbor     https://helm.goharbor.io
$ helm repo add  bitnami    https://charts.bitnami.com/bitnami
$ helm repo add  incubator  https://charts.helm.sh/incubator
$ helm repo add  stable     https://charts.helm.sh/stable
```

增加完仓库后，需要执行更新命令，将仓库中的信息进行同步：

```bash
$ helm repo update
```

> 注意：如果有的仓库不能正常解析，请更换 DNS 地址，在测试过程中，发现有的能正常解析，有的不能。如果还不行，就直接将域名和对应的地址写死在 Host 文件中

## Helm 命令

1）通过 Helm 在 Repo 中查询可安装的 Nginx 包：

```bash
$ helm search repo nginx
NAME                            	CHART VERSION	APP VERSION	DESCRIPTION                                       
bitnami/nginx                   	8.9.1        	1.19.10    	Chart for the nginx server                        
bitnami/nginx-ingress-controller	7.6.9        	0.46.0     	Chart for the nginx Ingress controller            
bitnami/kong                    	3.7.4        	2.4.1      	Kong is a scalable, open source API layer (aka ...
```

**安装测试：**

```bash
# 安装并查看应用状态
$ helm install nginx bitnami/nginx -n ns-test
$ helm status nginx -n ns-test
```

## Helm 参数

Helm 中支持使用自定义 `yaml` 文件和 `--set` 命令参数对要安装的应用进行参数配置，使用如下：

**查看应用 chart 可配置参数**

首先使用 helm show values {仓库名称}/{应用名称} 来查看对应应用的可配置参数：

```yaml
$ helm show values bitnami/nginx

image:
  registry: docker.io
  repository: bitnami/nginx
  tag: 1.17.10-debian-10-r33
resources:
  limits: 
     cpu: 100m
     memory: 128Mi
  requests: 
     cpu: 100m
     memory: 128Mi
  ......(太长,略)
```

- 方法一：使用自定义 values.yaml 配置文件安装应用，指定参数：

```yaml
$ cat > values.yaml << EOF

image:
  registry: docker.io
  repository: bitnami/nginx
resources:
  limits: 
     cpu: 1000m
     memory: 1024Mi
  requests: 
     cpu: 1000m
     memory: 1024Mi

EOF

# 使用自定义配置文件运行应用
$ helm install -f values.yaml bitnami/nginx --generate-name
```

- 方法二：使用 --set 配置参数进行安装，--set 参数是在使用 helm 命令时候添加的参数，可以在执行 helm 安装与更新应用时使用，多个参数间用","隔开，使用如下：

> 如果配置文件和 --set 同时使用，则 --set 设置的参数会覆盖配置文件中的参数配置

```bash
$ helm install --set 'registry.registry=docker.io,registry.repository=bitnami/nginx' bitnami/nginx
```

**参考解义**：对于 --set 写配置参数，**Helm 官方对于不同的配置类型给出了不同的写法**，如下：

| yaml 文件写法                             | set 的写法                                       |
| ----------------------------------------- | ------------------------------------------------ |
| name: value                               | --set name=value                                 |
| a: b c: d                                 | --set a=b,c=d                                    |
| outer:  inner: value                      | --set outer.inner=value                          |
| name:  - a  - b  - c                      | --set name={a, b, c}                             |
| servers:  - port: 80                      | --set servers[0].port=80                         |
| servers:  - port: 80   host: example      | --set servers[0].port=80,servers[0].host=example |
| name: "value1,value2"                     | --set name=value1\,value2                        |
| nodeSelector:  kubernetes.io/role: master | --set nodeSelector."kubernetes\.io/role"=master  |

## 卸载应用

1）卸载应用，并保留安装记录

```bash
$ helm uninstall nginx -n ns-test --keep-history
```

2）查看全部应用（包含安装和卸载的应用）

```bash
$ helm list -n ns-test --all
```

3）卸载应用，不保留安装记录

```bash
$ helm delete nginx -n ns-test
```

## 升级应用

1）创建新的配置参数文件 values.yaml：

```bash
$ cat > values.yaml << EOF

service.type: NodePort
service.nodePorts.http: 30002

EOF

# 应用更新
$ helm upgrade -f values.yaml nginx bitnami/nginx -n ns-test
```

2）查看新配置是否生效

```bash
$ helm get values nginx -n ns-test

USER-SUPPLIED VALUES:
service.nodePorts.http: 30002
service.type: NodePort
```

## 应用回滚

1）如果升级过程发生错误，进行回滚，首先查看应用的历史版本：

```bash
$ helm history nginx -n ns-test
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
1               Fri May  8 06:46:56 2020        superseded      nginx-5.3.1     1.17.3          Install complete
2               Fri May  8 06:46:56 2020        deployed        nginx-5.3.1     1.17.3          Upgrade complete
```

2）知道 REVISION 号后就可以进行回滚操作：

```bash
$ helm rollback nginx 1 -n ns-test
Rollback was a success! Happy Helming!
```

## 渲染模板

如果想查看通过指定的参数渲染的 Kubernetes 部署资源模板，可以通过下面命令：

```yaml
$ helm template bitnami/nginx -n ns-test
---
# Source: nginx/templates/svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: RELEASE-NAME-nginx
  labels:
    app.kubernetes.io/name: nginx
    helm.sh/chart: nginx-5.3.1
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
spec:
  type: LoadBalancer
  externalTrafficPolicy: "Cluster"
  ports:
    - name: http
      port: 80
      targetPort: http
  selector:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/instance: RELEASE-NAME
---
# Source: nginx/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: RELEASE-NAME-nginx
  labels:
    app.kubernetes.io/name: nginx
    helm.sh/chart: nginx-5.3.1
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
      app.kubernetes.io/instance: RELEASE-NAME
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
        helm.sh/chart: nginx-5.3.1
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
    spec:
      containers:
        - name: nginx
          image: docker.io/bitnami/nginx:1.17.3-debian-10-r63
          imagePullPolicy: "IfNotPresent"
          ports:
            - name: http
              containerPort: 8080
```