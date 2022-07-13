# CentOS7.x内核升级

> CentOS7.x 安装后，系统默认的内核版本是 3.10.0。过于旧的内核可能已经不再维护或会有安全风险，并且在部署个别环境&程序，会有不兼容的情况，建议提前进行升级。
> **注意：更新内核是有风险的，在操作之前慎重，严谨在生产环境上操作**

## 线上安装

1)安装 EL 源

```bash
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
$ rpm -Uvh https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
```

2)查看现可提供升级的版本（长期维护版本为 lt，主线最新版本为 ml）

```bash
$ yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * elrepo-kernel: hkg.mirror.rackspace.com
Available Packages
elrepo-release.noarch                          7.0-5.el7.elrepo                  elrepo-kernel
kernel-lt-devel.x86_64                         5.4.182-1.el7.elrepo              elrepo-kernel
kernel-lt-doc.noarch                           5.4.182-1.el7.elrepo              elrepo-kernel
kernel-lt-headers.x86_64                       5.4.182-1.el7.elrepo              elrepo-kernel
kernel-lt-tools.x86_64                         5.4.182-1.el7.elrepo              elrepo-kernel
kernel-lt-tools-libs.x86_64                    5.4.182-1.el7.elrepo              elrepo-kernel
kernel-lt-tools-libs-devel.x86_64              5.4.182-1.el7.elrepo              elrepo-kernel
kernel-ml.x86_64                               5.16.12-1.el7.elrepo              elrepo-kernel
kernel-ml-devel.x86_64                         5.16.12-1.el7.elrepo              elrepo-kernel
kernel-ml-doc.noarch                           5.16.12-1.el7.elrepo              elrepo-kernel
kernel-ml-headers.x86_64                       5.16.12-1.el7.elrepo              elrepo-kernel
kernel-ml-tools.x86_64                         5.16.12-1.el7.elrepo              elrepo-kernel
kernel-ml-tools-libs.x86_64                    5.16.12-1.el7.elrepo              elrepo-kernel
kernel-ml-tools-libs-devel.x86_64              5.16.12-1.el7.elrepo              elrepo-kernel
perf.x86_64                                    5.16.12-1.el7.elrepo              elrepo-kernel
python-perf.x86_64                             5.16.12-1.el7.elrepo              elrepo-kernel
```

3)选择版本进行安装

```bash
$ yum -y install kernel-ml --enablerepo=elrepo-kernel
```

## 离线安装

> 实际生产环境可能需要指定内核版本，并且无法联网直接安装，需提前下载 rpm 包进行安装
> [Centos内核rpm包下载地址](http://elrepo.org/linux/kernel/el7/x86_64/RPMS/)

```bash
$ wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-lt-5.4.182-1.el7.elrepo.x86_64.rpm
$ wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-lt-devel-5.4.182-1.el7.elrepo.x86_64.rpm

$ rpm -ivh kernel-lt-5.4.182-1.el7.elrepo.x86_64.rpm
$ rpm -ivh kernel-lt-devel-5.4.182-1.el7.elrepo.x86_64.rpm
$ yum localinstall -y kernel-lt*
```

## 配置内核

> 内核安装好后，需要将默认的启动选项改成新安装的，并重启方可生效

1)查看现有的内核版本

```bash
$ awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
0 : CentOS Linux (5.4.182-1.el7.elrepo.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1160.36.2.el7.x86_64) 7 (Core)
2 : CentOS Linux (3.10.0-1160.el7.x86_64) 7 (Core)
3 : CentOS Linux (0-rescue-678752f2ad574ce2b3d58a01705c580f) 7 (Core)
```

2)设置 `GRUB_DEFAULT=0`，并重启服务器

```bash
$ grub2-set-default 0
$ vim /etc/default/grub
# 修改索引号为0
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos_data/root rd.lvm.lv=centos_data/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"

$ grub2-mkconfig -o /boot/grub2/grub.cfg && reboot now
# 重启后验证内核信息
$ uname -a
Linux update 5.4.182-1.el7.elrepo.x86_64 #1 SMP Tue Mar 1 13:06:47 EST 2022 x86_64 x86_64 x86_64 GNU/Linux
```

## 卸载旧核

> 非必要，可自行选择

```bash
$ rpm -qa | grep kernel
$ yum -y remove $(rpm -qa | grep kernel | grep '3')
```
