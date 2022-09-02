# Linux Issue

> 主要记录一些特殊的问题

## 删除乱码文件

当文件名为乱码的时候，无法通过 rm/mv 等命令进行处理，此时可利用 i 节点号来操作

1）获取文件的 id 号

```bash
$ ls -li
total 11436
 67436148 drwxrwxr-x. 15 root  root      4096 Aug 31 11:37 cmake-3.24.1
101262465 -rw-r--r--.  1 root  root  10392868 Aug 18 04:35 cmake-3.24.1.tar.gz
 67442095 drwxr-xr-x.  9 mysql mysql      186 Aug 31 11:02 nginx-1.22.0
100676873 -rw-r--r--.  1 root  root   1073322 May 24 22:29 nginx-1.22.0.tar.gz
 67498140 drwxr-xr-x.  5 nginx users     4096 Aug 31 12:03 patchelf-0.15.0
101976868 -rw-r--r--.  1 root  root    226065 Jul 16 13:28 patchelf-0.15.0.tar.gz
102023683 -rw-r--r--.  1 root  root         0 Aug 31 12:05 z??ˁ?}???e?
```

2）执行删除指定的 id 号文件

```bash
$ find . -inum 102023683 -exec rm {} \;
```

## 重启磁盘名称错乱

服务器挂载了多个硬盘，重启后所挂载的 `/dev/sd*` 错乱，导致系统引导失败

**原因**：是 SCSI 子系统将 Linux 中的设备扫描计划为异步进行，设备路径名在重新启动时可能会有所不同

**解决方法**：对 Linux VM 使用文件系统标签或 UUID

1）先查看现有磁盘的 UUID
```bash
# 确认新磁盘 /dev/sda 的 UUID
$ sudo blkid -s UUID
/dev/sda: UUID="4a55017c-3639-4388-b794-f189ccfa6e76" 
/dev/sdb1: UUID="12271491-d64a-4032-8946-6edbdac0760a" 
/dev/sdb2: UUID="8V10ZR-Z2K0-pJps-7gpe-3eSf-3lQy-7c0YEC" 
/dev/sr0: UUID="2020-11-03-14-55-29-00" 
/dev/mapper/centos-root: UUID="b73b8b30-452c-4b7d-829e-304f2db46190" 
/dev/mapper/centos-swap: UUID="128253d9-f029-4a94-b6f9-4cbc96df309d"
```

2）以 UUID 方式添加新硬盘并重新加载

```bash
$ sudo vim /etc/fstab
UUID=4a55017c-3639-4388-b794-f189ccfa6e76 /mnt/ex-storage         ext4    defaults        0 0
$ sudo mount -a
```