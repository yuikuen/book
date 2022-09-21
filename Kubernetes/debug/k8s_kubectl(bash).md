# K8s Kubectl(Bash)

> kubectl 是 kubernetes 集群的命令行管理工具，在运维与部署应用时需要大量使用到 kubectl 命令对集群进行资源创建，日志查看等等操作

通过 [bash-completion](https://caliban.org/bash/) 的脚本实现可编程的补全程序，减少系统管理员日常维护工作，减少差错，提高工作效率

1）安装 bash-completion

```bash
$ yum install -y bash-completion 
$ source /usr/share/bash-completion/bash_completion
```

2）应用 kubectl 的 completion 到系统环境

```bash
$ source <(kubectl completion bash)
$ echo "source <(kubectl completion bash)" >> ~/.bashrc
```

**输入命令，双击 Tab 即可快速查看相关命令**

```bash
$ kubectl get 
apiservices.apiregistration.k8s.io                            felixconfigurations.crd.projectcalico.org                     nodes.metrics.k8s.io
bgpconfigurations.crd.projectcalico.org                       flowschemas.flowcontrol.apiserver.k8s.io                      persistentvolumeclaims
bgppeers.crd.projectcalico.org                                globalnetworkpolicies.crd.projectcalico.org                   persistentvolumes
blockaffinities.crd.projectcalico.org                         globalnetworksets.crd.projectcalico.org                       poddisruptionbudgets.policy
caliconodestatuses.crd.projectcalico.org                      horizontalpodautoscalers.autoscaling                          pods
certificatesigningrequests.certificates.k8s.io                hostendpoints.crd.projectcalico.org                           podsecuritypolicies.policy
clusterinformations.crd.projectcalico.org                     ingressclasses.networking.k8s.io                              pods.metrics.k8s.io
clusterrolebindings.rbac.authorization.k8s.io                 ingresses.networking.k8s.io                                   podtemplates
clusterroles.rbac.authorization.k8s.io                        ipamblocks.crd.projectcalico.org                              priorityclasses.scheduling.k8s.io
componentstatuses                                             ipamconfigs.crd.projectcalico.org                             prioritylevelconfigurations.flowcontrol.apiserver.k8s.io
configmaps                                                    ipamhandles.crd.projectcalico.org                             replicasets.apps
controllerrevisions.apps                                      ippools.crd.projectcalico.org                                 replicationcontrollers
cronjobs.batch                                                ipreservations.crd.projectcalico.org                          resourcequotas
csidrivers.storage.k8s.io                                     jobs.batch                                                    rolebindings.rbac.authorization.k8s.io
csinodes.storage.k8s.io                                       kubecontrollersconfigurations.crd.projectcalico.org           roles.rbac.authorization.k8s.io
csistoragecapacities.storage.k8s.io                           leases.coordination.k8s.io                                    runtimeclasses.node.k8s.io
customresourcedefinitions.apiextensions.k8s.io                limitranges                                                   secrets
daemonsets.apps                                               mutatingwebhookconfigurations.admissionregistration.k8s.io    serviceaccounts
deployments.apps                                              namespaces                                                    services
endpoints                                                     networkpolicies.crd.projectcalico.org                         statefulsets.apps
endpointslices.discovery.k8s.io                               networkpolicies.networking.k8s.io                             storageclasses.storage.k8s.io
events                                                        networksets.crd.projectcalico.org                             validatingwebhookconfigurations.admissionregistration.k8s.io
events.events.k8s.io                                          nodes                                                         volumeattachments.storage.k8s.io
```
