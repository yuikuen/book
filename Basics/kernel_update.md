# Kernel Update

> 过旧的内核可能已不再维护或有安全漏洞等问题，并且部署环境/程序也有不兼容的情况，建议按需升级
> 
> 注意：**更新内核是有风险的，在操作之前慎重，严谨在生产环境上操作**

## YUM 源安装

1）安装 EL 源

```bash
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
$ rpm -Uvh https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
Retrieving https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:elrepo-release-7.0-6.el7.elrepo  ################################# [100%
```

2）查看现可提供升级的版本

> lt 为长期维护版本，ml 为主线最新版本

```bash
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * elrepo-kernel: mirror.rackspace.com
Available Packages
kernel-lt.x86_64                                 5.4.207-1.el7.elrepo                elrepo-kernel
kernel-lt-devel.x86_64                           5.4.207-1.el7.elrepo                elrepo-kernel
kernel-lt-doc.noarch                             5.4.207-1.el7.elrepo                elrepo-kernel
kernel-lt-headers.x86_64                         5.4.207-1.el7.elrepo                elrepo-kernel
kernel-lt-tools.x86_64                           5.4.207-1.el7.elrepo                elrepo-kernel
kernel-lt-tools-libs.x86_64                      5.4.207-1.el7.elrepo                elrepo-kernel
kernel-lt-tools-libs-devel.x86_64                5.4.207-1.el7.elrepo                elrepo-kernel
kernel-ml-doc.noarch                             5.18.14-1.el7.elrepo                elrepo-kernel
kernel-ml-headers.x86_64                         5.18.14-1.el7.elrepo                elrepo-kernel
kernel-ml-tools.x86_64                           5.18.14-1.el7.elrepo                elrepo-kernel
kernel-ml-tools-libs.x86_64                      5.18.14-1.el7.elrepo                elrepo-kernel
kernel-ml-tools-libs-devel.x86_64                5.18.14-1.el7.elrepo                elrepo-kernel
perf.x86_64                                      5.18.14-1.el7.elrepo                elrepo-kernel
python-perf.x86_64                               5.18.14-1.el7.elrepo                elrepo-kerne
```

3）选择版本进行安装

```bash
$ yum -y install kernel-ml --enablerepo=elrepo-kernel
```

## RPM 源安装

> 实际生产环境一般需要指定内核版本，并且无法联网安装，这时需提前下载 [RPM 包](http://elrepo.org/linux/kernel/el7/x86_64/RPMS/) 进行安装

1）到 [ELREPO](https://elrepo.org/linux/kernel/) 下载自身需要的 RPM 包

```bash
$ wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-ml-5.18.14-1.el7.elrepo.x86_64.rpm
$ wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-ml-devel-5.18.14-1.el7.elrepo.x86_64.rpm
```

2）直接加载安装

```bash
$ ll
total 74116
-rw-r--r--. 1 root root 61101152 Jul 26 16:18 kernel-ml-5.18.14-1.el7.elrepo.x86_64.rpm
-rw-r--r--. 1 root root 14789108 Jul 26 16:16 kernel-ml-devel-5.18.14-1.el7.elrepo.x86_64.rpm

$ rpm -ivh kernel-ml-5.18.14-1.el7.elrepo.x86_64.rpm kernel-ml-devel-5.18.14-1.el7.elrepo.x86_64.rpm
warning: kernel-ml-5.18.14-1.el7.elrepo.x86_64.rpm: Header V4 DSA/SHA256 Signature, key ID baadae52: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:kernel-ml-devel-5.18.14-1.el7.elr################################# [ 50%]
   2:kernel-ml-5.18.14-1.el7.elrepo   ################################# [100%]
```

## 配置内核

> 安装后需要将默认的启动选项改成新安装的，并重启方可生效

1）查看现有内核版本

```bash
$ awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
0 : CentOS Linux (5.18.14-1.el7.elrepo.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1160.el7.x86_64) 7 (Core)
2 : CentOS Linux (0-rescue-77ada07bc97843b3a8e034039509ee14) 7 (Core)
```

2）将序号 0 的设置为密码的启动项并重新加载、重启服务器

```bash
$ grub2-set-default 0
$ grub2-mkconfig -o /boot/grub2/grub.cfg && reboot now
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.18.14-1.el7.elrepo.x86_64
Found initrd image: /boot/initramfs-5.18.14-1.el7.elrepo.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-1160.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1160.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-4d7bc3c723f2437d995ca26258759d4e
Found initrd image: /boot/initramfs-0-rescue-4d7bc3c723f2437d995ca26258759d4e.img
done
```

## 卸载旧核 

> 非必要，可自行选择

```bash
$ rpm -qa | grep kernel
$ yum -y remove $(rpm -qa | grep kernel | grep '3')
```