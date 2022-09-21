# Manjaro TimeShift

> 在 Window 系统中，可以借助微软自带的工具或 Dism++ 等工具进行系统备份，当系统出现故障时，可通过备份的快照来恢复系统。而在 Linux 下也有类似的工具，如 **TimeShift**、**Rsync**、**Fwbackups** 等等，在此来介绍 Manjaro 下较为好用的 **TimeShift**

```bash
$ sudo pacman -S timeshift
```

当成功安装后，桌面就会出现 `Timeshift` 的图标

1）打开软件会出现设备向导

一般选择 RSYNC，它支持增量备份，在 RSYNC 模式每次备份的时候，只传输改变的部分。而 BTRFS 模式下支持创建一个系统的完整快照，根据自身需求设置

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/2021-12-07_19-16.png)

2）选择备份文件存储的位置

设置快照放在哪里，因为当恢复系统的时候，需要去选择快照

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/2021-12-07_19-19.png)

3）选择备份时间

根据需要选择，其中"保留"后面的数字意思是：快照数量超过此数字时，会自动删除多余的快照

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/2021-12-07_19-19_1.png)

4）选择要备份的目录（默认即可）

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/2021-12-07_19-20.png)

