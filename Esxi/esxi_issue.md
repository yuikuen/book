# Esxi Issue

> 记录 Esxi 平台上遇到的问题

## 虚拟机关机卡住

> ESXi 在日常使用时经常会遇到机器卡住的情况 这种情况下 GUI 的方式无从下手, 需要从 cli 的方式处理

1）首先开启宿主机的 shell (SSH) 服务

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20210824102352466.png)

2）**Xshell 登录**，用户名 root 密码 就是安装时设置的 ESXi 的密码，可以用 esxtop 查看具体性能情况

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20210824102735270.png)

3）**查看虚拟机**并根据机器 id 进行关闭指定虚拟机

```bash
# 查看虚拟机的列表
$ esxcli vm process list 
Gitlab
   World ID: 2099532
   Process ID: 0
   VMX Cartel ID: 2099517
   UUID: 56 4d 48 08 86 8b 8f 67-a5 4a 33 f3 87 a8 d0 59
   Display Name: Gitlab
   Config File: /vmfs/volumes/6107a9c3-0511c860-9122-fcaa144d4e7d/Gitlab/Gitlab.vmx

Jenkins
   World ID: 2108579
   Process ID: 0
   VMX Cartel ID: 2108578
   UUID: 56 4d 7e f1 59 73 75 50-f4 90 c8 4f 61 20 43 7a
   Display Name: Jenkins
   Config File: /vmfs/volumes/6107a9c3-0511c860-9122-fcaa144d4e7d/Jenkins/Jenkins.vmx

dev
   World ID: 2166668
   Process ID: 0
   VMX Cartel ID: 2166667
   UUID: 56 4d ec fe f6 5f 03 80-95 97 de 30 60 ff 71 9c
   Display Name: dev
   Config File: /vmfs/volumes/6107a9f1-67821e71-248c-fcaa144d4e7d/dev/dev.vmx

harbor&nexus
   World ID: 2206805
   Process ID: 0
   VMX Cartel ID: 2206804
   UUID: 56 4d 26 c0 b4 1b 45 fd-b4 6d e6 ca 1e 64 30 a8
   Display Name: harbor&nexus
   Config File: /vmfs/volumes/6107a9c3-0511c860-9122-fcaa144d4e7d/harbor&nexus/harbor&nexus.vmx
```

根据上面查询出来的 world id 直接进行下一步 `esxcli vm process kill --world-id= `

```bash
# 根据虚拟机的名字查找机器world id
$ esxcli vm process list | grep jenkins

# 根据world id 强制关闭虚拟机.
$ esxcli vm process kill -w=2108579 -t=hard
```

之后再查看列表是否已经删除并正常关机了