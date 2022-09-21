# K8s_Calico Bug

> 异常问题记录和解决方案

## 问题一

`calico-node 反复重启，状态为 CrashBackoff，错误提示 Failed to reach apiserver error=<nil>`

**问题原因**

Node 工作节点连接不到 `apiserver` 地址

**解决方法**

Calico 是通过 Kubernetes yaml 文件部署的，直接在 yaml 文件中 `DaemonSet` 添加配置

```bash
376             - name: FELIX_HEALTHENABLED
377               value: "true"
                # 在calico-node的DaemonSet中添加env环境变量参数
                - name: KUBERNETES_SERVICE_HOST
                  value: "188.188.4.120"  # master apiserver地址
                - name: KUBERNETES_SERVICE_PORT
                  value: "6443"
                - name: KUBERNETES_SERVICE_PORT_HTTPS
                  value: "6443"
```

## 问题二

`calico-kube-controllers 一直重启，状态为 CrashBackoff，错误提示 Failed to reach apiserver error=<nil>` 

**问题原因**

与 calico-node 同理

**解决方法**

需要在 yaml 文件的 `Deployment` 中添加 env 参数项

```bash
558             # Choose which controllers to run.
559             - name: ENABLED_CONTROLLERS
560               value: policy,namespace,serviceaccount,workloadendpoint,node
                # 在calico-kube-controllers的Deployment中添加env环境变量参数
                - name: KUBERNETES_SERVICE_HOST
                  value: "188.188.4.120"  # master apiserver地址
                - name: KUBERNETES_SERVICE_PORT
                  value: "6443"
                - name: KUBERNETES_SERVICE_PORT_HTTPS
                  value: "6443"          
```

## 问题三

`Readiness probe failed: caliconode is not ready: BIRD is not ready: BGP not established with 172.16.0.1`

**问题原因**

Calico 默认会自动识别第一个网卡，IP 识别策略(IPDETECTMETHOD)未配置，默认为 first-found，导致网络异常的 ip 作为 nodeIP 被注册，从而影响 node-to-node mesh

**解决方法**

通过修改 yaml 文件中的配置，定义网卡发现规则

```bash
[root@k8s-master01 calico]# ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:c2:47:c8 brd ff:ff:ff:ff:ff:ff
    inet 188.188.4.121/24 brd 188.188.4.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 188.188.4.120/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fec2:47c8/64 scope link 
       valid_lft forever preferred_lft forever
...

$ vim calico.yaml +4332
4330             # Cluster type to identify the deployment type
4331             - name: CLUSTER_TYPE
4332               value: "k8s,bgp"
                 # 定义ipv4自动发现网卡规则，根据自身网卡，此处为eth0
                 - name: IP_AUTODETECTION_METHOD
                   value: "interface=eth.*"
```

