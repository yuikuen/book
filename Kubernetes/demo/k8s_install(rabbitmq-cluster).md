# K8s Install(RabbitMQ-Cluster)

**RabbitMQ** 是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件）

- 可伸缩性：集群服务
- 消息持久化：从内存持久化消息到硬盘，再从硬盘加载到内存 

下述教程就是采用 Kubernetes 来安装 RabbitMQ，并且通过 `rabbitmq_management,rabbitmq_peer_discovery_k8s` 来自动发现并注册集群，而 RabbitMQ 使用 StatefulSet 来部署，通过域名来访问，方便后面统一管理

- Kubernetes 集群
- 持久化数据
- [Yaml 参考文件](https://github.com/dotbalo/k8s/)

1）创建 namespace

```bash
$ kubectl create namespace public-service
# 如果不使用public-service，需要更改所有yaml文件的public-service为你namespace
$ sed -i "s#public-service#YOUR_NAMESPACE#g" *.yaml
```

2）创建 rbac

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rmq-cluster
  namespace: public-service
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rmq-cluster
  namespace: public-service
rules:
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rmq-cluster
  namespace: public-service
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rmq-cluster
subjects:
- kind: ServiceAccount
  name: rmq-cluster
  namespace: public-service
```

3）创建 secret

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: rmq-cluster-secret
  namespace: public-service
stringData:
  cookie: ERLANG_COOKIE
  password: RABBITMQ_PASS
  url: amqp://RABBITMQ_USER:RABBITMQ_PASS@rmq-cluster-balancer
  username: RABBITMQ_USER
type: Opaque
```

4）创建 configmap

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: rmq-cluster-config
  namespace: public-service
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
    enabled_plugins: |
      [rabbitmq_management,rabbitmq_peer_discovery_k8s].
    rabbitmq.conf: |
      loopback_users.guest = false
      # 因采用secret加密方式，需要添加账密字段
      default_user = RABBITMQ_USER
      default_pass = RABBITMQ_PASS
      ## Clustering
      cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      cluster_formation.k8s.address_type = hostname
      #################################################
      # public-service is rabbitmq-cluster's namespace#
      #################################################
      cluster_formation.k8s.hostname_suffix = .rmq-cluster.public-service.svc.cluster.local
      cluster_formation.node_cleanup.interval = 10
      cluster_formation.node_cleanup.only_log_warning = true
      cluster_partition_handling = autoheal
      ## queue master locator
      queue_master_locator=min-masters
```

5）部署应用及服务(此处未作持久化，如需可删除 pv 注释)

```yaml
# 此处采用的是静态PV方式，后端使用的是NFS，为了方便扩展可以使用动态PV较好
kind: StatefulSet
apiVersion: apps/v1
metadata:
  labels:
    app: rmq-cluster
  name: rmq-cluster
  namespace: public-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rmq-cluster
  serviceName: rmq-cluster
  template:
    metadata:
      labels:
        app: rmq-cluster
    spec:
      containers:
      - args:
        - -c
        - cp -v /etc/rabbitmq/rabbitmq.conf ${RABBITMQ_CONFIG_FILE}; exec docker-entrypoint.sh
          rabbitmq-server
        command:
        - sh
        env:
        - name: RABBITMQ_DEFAULT_USER
          valueFrom:
            secretKeyRef:
              key: username
              name: rmq-cluster-secret
        - name: RABBITMQ_DEFAULT_PASS
          valueFrom:
            secretKeyRef:
              key: password
              name: rmq-cluster-secret
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              key: cookie
              name: rmq-cluster-secret
        - name: K8S_SERVICE_NAME
          value: rmq-cluster
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: RABBITMQ_USE_LONGNAME
          value: "true"
        - name: RABBITMQ_NODENAME
          value: rabbit@$(POD_NAME).rmq-cluster.$(POD_NAMESPACE).svc.cluster.local
        - name: RABBITMQ_CONFIG_FILE
          value: /var/lib/rabbitmq/rabbitmq.conf
        image: rabbitmq:3.9.5-management
        imagePullPolicy: IfNotPresent
        livenessProbe:
          exec:
            command:
            - rabbitmqctl
            - status
          initialDelaySeconds: 30
          timeoutSeconds: 10
        name: rabbitmq
        ports:
        - containerPort: 15672
          name: http
          protocol: TCP
        - containerPort: 5672
          name: amqp
          protocol: TCP
        readinessProbe:
          exec:
            command:
            - rabbitmqctl
            - status
          initialDelaySeconds: 10
          timeoutSeconds: 10
        volumeMounts:
        - mountPath: /etc/rabbitmq
          name: config-volume
          readOnly: false
#        - mountPath: /var/lib/rabbitmq
#          name: rabbitmq-storage
#          readOnly: false
      serviceAccountName: rmq-cluster
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          items:
          - key: rabbitmq.conf
            path: rabbitmq.conf
          - key: enabled_plugins
            path: enabled_plugins
          name: rmq-cluster-config
        name: config-volume
#  volumeClaimTemplates:
#  - metadata:
#      name: rabbitmq-storage
#    spec:
#      accessModes:
#      - ReadWriteMany
#      storageClassName: "rmq-storage-class"
#      resources:
#        requests:
#          storage: 4Gi
```

```yaml
# rabbitmq svc服务
kind: Service
apiVersion: v1
metadata:
  labels:
    app: rmq-cluster
  name: rmq-cluster
  namespace: public-service
spec:
  clusterIP: None
  ports:
  - name: amqp
    port: 5672
    targetPort: 5672
  selector:
    app: rmq-cluster
```

```yaml
# rabbitmq lb，用于代理svc
kind: Service
apiVersion: v1
metadata:
  labels:
    app: rmq-cluster
    type: LoadBalancer
  name: rmq-cluster-balancer
  namespace: public-service
spec:
  ports:
  - name: http
    port: 15672
    protocol: TCP
    targetPort: 15672
  - name: amqp
    port: 5672
    protocol: TCP
    targetPort: 5672
  selector:
    app: rmq-cluster
  type: NodePort
```

6）创建 PV (参考项：如作持久化可作参考)

```yaml
$ cat rabbitmq-pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    # real share directory
    path: /ifs/kubernetts/rabbitmq-cluster-0
    # nfs real ip
    server: 188.188.3.110

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-2
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    # real share directory
    path: /ifs/kubernetts/rabbitmq-cluster-1
    # nfs real ip
    server: 188.188.3.110

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rmq-3
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: "rmq-storage-class"
  nfs:
    # real share directory
    path: /ifs/kubernetts/rabbitmq-cluster-2
    # nfs real ip
    server: 188.188.3.110
```

7）创建集群

```bash
$ kubectl apply -f .
statefulset.apps/rmq-cluster created
configmap/rmq-cluster-config created
persistentvolume/pv-rmq-1 created
persistentvolume/pv-rmq-2 created
persistentvolume/pv-rmq-3 created
serviceaccount/rmq-cluster created
role.rbac.authorization.k8s.io/rmq-cluster created
rolebinding.rbac.authorization.k8s.io/rmq-cluster created
secret/rmq-cluster-secret created
service/rmq-cluster created
service/rmq-cluster-balancer created
```

8）查看状态

```bash
$ kubectl get pod,sts,svc,ep,pv,pvc -n public-service 
NAME                READY   STATUS    RESTARTS   AGE
pod/rmq-cluster-0   1/1     Running   0          8m10s
pod/rmq-cluster-1   1/1     Running   0          7m34s
pod/rmq-cluster-2   1/1     Running   0          7m18s

NAME                           READY   AGE
statefulset.apps/rmq-cluster   3/3     8m11s

NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                          AGE
service/rmq-cluster            ClusterIP   None           <none>        5672/TCP                         8m11s
service/rmq-cluster-balancer   NodePort    10.96.58.102   <none>        15672:31824/TCP,5672:31993/TCP   8m11s

NAME                             ENDPOINTS                                                              AGE
endpoints/rmq-cluster            10.244.32.129:5672,10.244.58.225:5672,10.244.85.246:5672               8m11s
endpoints/rmq-cluster-balancer   10.244.32.129:5672,10.244.58.225:5672,10.244.85.246:5672 + 3 more...   8m11s

NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                           STORAGECLASS        REASON   AGE
persistentvolume/pv-rmq-1   1Gi        RWX            Recycle          Bound    public-service/rabbitmq-storage-rmq-cluster-2   rmq-storage-class            8m11s
persistentvolume/pv-rmq-2   1Gi        RWX            Recycle          Bound    public-service/rabbitmq-storage-rmq-cluster-1   rmq-storage-class            8m11s
persistentvolume/pv-rmq-3   1Gi        RWX            Recycle          Bound    public-service/rabbitmq-storage-rmq-cluster-0   rmq-storage-class            8m11s
persistentvolume/pv0001     2Gi        RWO            Recycle          Bound    default/test-pvc2                               slow                         9d

NAME                                                   STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
persistentvolumeclaim/rabbitmq-storage-rmq-cluster-0   Bound    pv-rmq-3   1Gi        RWX            rmq-storage-class   8m11s
persistentvolumeclaim/rabbitmq-storage-rmq-cluster-1   Bound    pv-rmq-2   1Gi        RWX            rmq-storage-class   7m34s
persistentvolumeclaim/rabbitmq-storage-rmq-cluster-2   Bound    pv-rmq-1   1Gi        RWX            rmq-storage-class   7m18s
```

可以通过 svc 的 nodeport 端口暴露，或者 ingress 来访问 rabbitmq 可视化界面

9）集群扩缩容

```bash
$ kubectl scale statefulset -n public-service --replicas=4 rmq-cluster
```