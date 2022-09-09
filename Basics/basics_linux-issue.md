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

## 删除时忽略某特定文件

使用命令在删除文件时排除或忽略某特定文件

1）通过开启通配符的功能进行删除

```bash
# 开启通配符功能，并查看是否开启，例子 rm -f !(name.file)
$ shopt -s extglob
$ shopt -s
checkwinsize   	on
cmdhist        	on
expand_aliases 	on
extglob        	on
extquote       	on
force_fignore  	on
histappend     	on
hostcomplete   	on
interactive_comments	on
login_shell    	on
progcomp       	on
promptvars     	on
sourcepath     	on

$ rm -rf !(1.tt)
```

2）通过执行 find 的结果，传至 xargs 进行转换后再交给 rm 命令处理

> 删除当前目录除名字为 1.tt 的所有文件

```bash
# 方法不一样，但执行结果一致
$ find . -not -name "1.tt" -exec rm -rf {} \;
$ find . -not -name "1.tt" | xargs rm -rf
```

3）或编写 shell 脚本，然后执行删除

```bash
$ for i in `ls`;do if [ "$i" != 1.tt ];then rm -rf $i;fi;done;
```

## 清除 buff & Cache

1）查看内存的使用情况

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:            31G         16G        1.1G        107M         13G         14G
Swap:           63G        6.0G         58G
```

- total：总内存
- used：已用内存
- free：空闲内存
- buff/cache：已使用的缓存
- avaiable：已用内存

2）执行 sync 命令清理缓存

执行 sync 命令是为了确保文件系统的完整性，手动执行 sync 命令，将所有未写的系统缓冲区写到磁盘中，包含已修改的 i-node、已延迟的块 I/O 和读写映射文件

```bash
# 表示清除pagecache
$ echo 1 > /proc/sys/vm/drop_caches

# 表示清除回收slab分配器中的对象（包括目录项缓存和inode缓存）。
# slab分配器是内核中管理内存的一种机制，其中很多缓存数据实现都是用的pagecache
$ echo 2 > /proc/sys/vm/drop_caches  

# 表示清除pagecache和slab分配器中的缓存对象
$ echo 3 > /proc/sys/vm/drop_caches
```

- Buffer 指的是 Linux 内存的：Buffer Cache (缓冲区缓存)
- Cache 指的是 Linux 内存的：Page Cache (页面缓存)

## 根目录扩容

> **业务场景**：服务器根目录容量不足，新增一块硬盘扩容到根目录

- PV(Physical Volume)- 物理卷
  
  物理卷在逻辑卷管理中处于最底层，它可以是实际物理硬盘上的分区，也可以是整个物理硬盘，也可以是raid设备

- VG(Volumne Group)- 卷组
  
  卷组建立在物理卷之上，一个卷组中至少要包括一个物理卷，在卷组建立之后可动态添加物理卷到卷组中。一个逻辑卷管理系统工程中可以只有一个卷组，也可以拥有多个卷组

- LV(Logical Volume)- 逻辑卷
  
  逻辑卷建立在卷组之上，卷组中的未分配空间可以用于建立新的逻辑卷，逻辑卷建立后可以动态地扩展和缩小空间。系统中的多个逻辑卷可以属于同一个卷组，也可以属于不同的多个卷组

1）查看当前根目录容量大小并确认新增加的磁盘信息

```bash
# 演示环境，模拟根目录占满35G
$ df -Th
Filesystem              Type      Size  Used Avail Use% Mounted on
devtmpfs                devtmpfs  3.9G     0  3.9G   0% /dev
tmpfs                   tmpfs     3.9G   28K  3.9G   1% /dev/shm
tmpfs                   tmpfs     3.9G   14M  3.9G   1% /run
tmpfs                   tmpfs     3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root xfs        35G  6.7G   29G  20% /
/dev/sdb1               xfs      1014M  187M  828M  19% /boot
tmpfs                   tmpfs     793M     0  793M   0% /run/user/0

# 新增一个大小为10G的磁盘，磁盘名为sda
$ fdisk -l
Disk /dev/sda: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

2）对新增加的磁盘进行分区

> 具体的磁盘操作可输入 m 获取帮助

```bash
$ fdisk /dev/sda 
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x47e2f96d.

# 创建并划分分区
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-20971519, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519): 
Using default value 20971519
Partition 1 of type Linux and of size 10 GiB is set

# 选择磁盘的类型
Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): p

Disk /dev/sda: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x47e2f96d

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            2048    20971519    10484736   8e  Linux LVM

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

3）创建物理卷并查看物理卷状态信息 vgdisplay

```bash
$ pvcreate /dev/sda1 
  Physical volume "/dev/sda1" successfully created.

# 查看物理卷信息
$ pvs
  PV         VG     Fmt  Attr PSize   PFree  
  /dev/sda1         lvm2 ---  <10.00g <10.00g
  /dev/sdb2  centos lvm2 a--  <39.00g      0 

# 查看较为详细的物理卷信息
$ pvscan
  PV /dev/sdb2   VG centos          lvm2 [<39.00 GiB / 0    free]
  PV /dev/sda1                      lvm2 [<10.00 GiB]
  Total: 2 [<49.00 GiB] / in use: 1 [<39.00 GiB] / in no VG: 1 [<10.00 GiB]

# 查看更加详细的物理卷信息
$ pvdisplay
  --- Physical volume ---
  PV Name               /dev/sdb2
  VG Name               centos
  PV Size               <39.00 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              9983
  Free PE               0
  Allocated PE          9983
  PV UUID               04e7AT-1uT0-2thu-IyON-N2ia-Y7mw-6YXZLW
   
  "/dev/sda1" is a new physical volume of "<10.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sda1
  VG Name               
  PV Size               <10.00 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               jpZn2p-lcOf-s5UI-GOTz-pifM-MQZn-grk0Rb
```

4）将新增加的分区 `/dev/sda1` 加入到根目录分区 centos (`VG Name` 卷组名称)

> 也可以使用 `vgdisplay` 查看卷组名称

```bash
$ vgextend centos /dev/sda1
  Volume group "centos" successfully extended

# 对根目录进行扩容
$ lvextend -l +100%FREE /dev/mapper/centos-root 
  Size of logical volume centos/root changed from <35.00 GiB (8959 extents) to 44.99 GiB (11518 extents).
  Logical volume centos/root successfully resized.

# 调整分区大小
$xfs_growfs /dev/mapper/centos-root 
meta-data=/dev/mapper/centos-root isize=512    agcount=4, agsize=2293504 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=9174016, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=4479, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 9174016 to 11794432
```

5）查看根目录大小，验证是否成功扩容

> 从原来的 30G 变成 40G

```bash
$ df -Th
Filesystem              Type      Size  Used Avail Use% Mounted on
devtmpfs                devtmpfs  3.9G     0  3.9G   0% /dev
tmpfs                   tmpfs     3.9G   28K  3.9G   1% /dev/shm
tmpfs                   tmpfs     3.9G   14M  3.9G   1% /run
tmpfs                   tmpfs     3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root xfs        45G  6.7G   39G  15% /
/dev/sdb1               xfs      1014M  187M  828M  19% /boot
tmpfs                   tmpfs     793M     0  793M   0% /run/user/0
```

## 修改远程端口

> Centos 7.x 安装好后，默认开启 ssh (22) 端口，和 root 远程访问权限，为了更安全的使用系统，最好是修改默认的远程端口并关闭 root 远程访问权限

1）修改 `/etc/ssh/sshd_config` 文件

```bash
$ vim /etc/ssh/sshd_config
# If you want to change the port on a SELinux system, you have to tell
# SELinux about this change.
# semanage port -a -t ssh_port_t -p tcp #PORTNUMBER
#
Port 22                //取消注释，使用默认的22端口
Port 65522             //增加新的远程端口65522
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
```

重启 sshd 服务

```bash
$ systemctl restart sshd.service
```

> 使用 65522 端口连接会发现，并不能连接终端设备，并且会报错误信息
>
> `Job for sshd.service failed because the control process exited with error code.
> See “systemctl status sshd.service” and “journalctl -xe” for details.`

按命令提示查看 sshd 状态发现，65522 端口绑定失败，没有生效

```bash
$ systemctl status sshd.service
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2022-02-10 15:59:21 CST; 16s ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 1854 (sshd)
   CGroup: /system.slice/sshd.service
           └─1854 /usr/sbin/sshd -D

Feb 10 15:59:21 nextcloud systemd[1]: Stopped OpenSSH server daemon.
Feb 10 15:59:21 nextcloud systemd[1]: Starting OpenSSH server daemon...
Feb 10 15:59:21 nextcloud sshd[1854]: error: Bind to port 65522 on 0.0.0.0 failed: Permission denied.
Feb 10 15:59:21 nextcloud sshd[1854]: error: Bind to port 65522 on :: failed: Permission denied.
Feb 10 15:59:21 nextcloud sshd[1854]: Server listening on 0.0.0.0 port 22.
Feb 10 15:59:21 nextcloud sshd[1854]: Server listening on :: port 22.
Feb 10 15:59:21 nextcloud systemd[1]: Started OpenSSH server daemon.
```

2）查看 SELinux 服务状态，并查看 ssh 服务端口绑定状态

```bash
$ /usr/sbin/sestatus -v
SELinux status:                 enabled             //状态为已开启
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31

$ semanage port -l | grep ssh
ssh_port_t                     tcp      22          //服务端口只有22，并没新绑定的65522
```

3）命令向 SELinux 添加 ssh 端口，并且查看相关服务状态 & 端口

```bash
$ semanage port -a -t ssh_port_t -p tcp 65522
$ semanage port -l | grep ssh
ssh_port_t                     tcp      65522, 22

$ systemctl restart sshd
$ systemctl status sshd.service
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2022-02-10 16:02:54 CST; 1s ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 1916 (sshd)
   CGroup: /system.slice/sshd.service
           └─1916 /usr/sbin/sshd -D

Feb 10 16:02:54 nextcloud systemd[1]: Starting OpenSSH server daemon...
Feb 10 16:02:54 nextcloud sshd[1916]: Server listening on 0.0.0.0 port 65522.
Feb 10 16:02:54 nextcloud sshd[1916]: Server listening on :: port 65522.
Feb 10 16:02:54 nextcloud sshd[1916]: Server listening on 0.0.0.0 port 22.
Feb 10 16:02:54 nextcloud systemd[1]: Started OpenSSH server daemon.
Feb 10 16:02:54 nextcloud sshd[1916]: Server listening on :: port 22.
```

重新查看 sshd 服务，发现此时 65522 端口已经在侦听状态

4）开启 firewall 端口

默认防火墙是阻止端口连接的，现 SSH 新端口也是无法连接的

```bash
$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens192
  sources: 
  services: dhcpv6-client ssh
  ports:                       //查看端口状态，未有开放的端口
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

```bash
$ firewall-cmd --zone=public --add-port=65522/tcp --permanent
$ firewall-cmd –reload
success
$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens192
  sources: 
  services: dhcpv6-client ssh
  ports: 65522/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

添加完规则后，重启载入使用其生效，此时再查看服务状态 ports 已多了 65522/tcp，现使用新端口远程连接，便可成功登陆系统

5）禁止 root 用户远程登录

```bash
$ vim /etc/ssh/sshd_config
# Authentication:

#LoginGraceTime 2m
PermitRootLogin no           //取消注释，并设置为no
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10
```

## 安装视频解码

> 为了 NextCloud 在线视频支持更多的视频源

-  方法一：源码编译安装

```bash
# 下载安装依赖
$ yum install gcc yasm
$ wget https://johnvansickle.com/ffmpeg/release-source/ffmpeg-4.1.tar.xz
$ tar -xf ffmpeg-4.1.tar.xz && cd ./ffmpeg-4.1

# 编译安装
$ ./configure --enable-shared --prefix=/usr/local/ffmpeg
$ make &&　make install
$ vim /etc/ld.so.conf
/usr/local/ffmpeg/lib/
$ ldconfig

# 配置环境变量
$ vim /etc/profile
PATH=$PATH:/usr/local/ffmpeg/bin
export PATH
$ source /etc/profile
$ ffmpeg -version
```

- 方法二：直接解压程序

```bash
# 解压至指定路径即可
$ wget https://johnvansickle.com/ffmpeg/builds/ffmpeg-git-amd64-static.tar.xz
$ tar -xf ffmpeg-git-amd64-static.tar.xz && cd ./ffmpeg-git-20220108-amd64-static
$ mv ffmpeg ffprobe /usr/bin/

# 验证版本
$ ffmpeg -version
ffmpeg version N-60236-gffb000fff8-static https://johnvansickle.com/ffmpeg/  Copyright (c) 2000-2022 the FFmpeg developers
built with gcc 8 (Debian 8.3.0-6)
configuration: --enable-gpl --enable-version3 --enable-static --disable-debug --disable-ffplay --disable-indev=sndio --disable-outdev=sndio --cc=gcc --enable-fontconfig --enable-frei0r --enable-gnutls --enable-gmp --enable-libgme --enable-gray --enable-libaom --enable-libfribidi --enable-libass --enable-libvmaf --enable-libfreetype --enable-libmp3lame --enable-libopencore-amrnb --enable-libopencore-amrwb --enable-libopenjpeg --enable-librubberband --enable-libsoxr --enable-libspeex --enable-libsrt --enable-libvorbis --enable-libopus --enable-libtheora --enable-libvidstab --enable-libvo-amrwbenc --enable-libvpx --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxml2 --enable-libdav1d --enable-libxvid --enable-libzvbi --enable-libzimg
libavutil      57. 18.100 / 57. 18.100
libavcodec     59. 20.100 / 59. 20.100
libavformat    59. 17.100 / 59. 17.100
libavdevice    59.  5.100 / 59.  5.100
libavfilter     8. 25.100 /  8. 25.100
libswscale      6.  5.100 /  6.  5.100
libswresample   4.  4.100 /  4.  4.100
libpostproc    56.  4.100 / 56.  4.100

$ ffprobe -version
ffprobe version N-60236-gffb000fff8-static https://johnvansickle.com/ffmpeg/  Copyright (c) 2007-2022 the FFmpeg developers
built with gcc 8 (Debian 8.3.0-6)
configuration: --enable-gpl --enable-version3 --enable-static --disable-debug --disable-ffplay --disable-indev=sndio --disable-outdev=sndio --cc=gcc --enable-fontconfig --enable-frei0r --enable-gnutls --enable-gmp --enable-libgme --enable-gray --enable-libaom --enable-libfribidi --enable-libass --enable-libvmaf --enable-libfreetype --enable-libmp3lame --enable-libopencore-amrnb --enable-libopencore-amrwb --enable-libopenjpeg --enable-librubberband --enable-libsoxr --enable-libspeex --enable-libsrt --enable-libvorbis --enable-libopus --enable-libtheora --enable-libvidstab --enable-libvo-amrwbenc --enable-libvpx --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxml2 --enable-libdav1d --enable-libxvid --enable-libzvbi --enable-libzimg
libavutil      57. 18.100 / 57. 18.100
libavcodec     59. 20.100 / 59. 20.100
libavformat    59. 17.100 / 59. 17.100
libavdevice    59.  5.100 / 59.  5.100
libavfilter     8. 25.100 /  8. 25.100
libswscale      6.  5.100 /  6.  5.100
libswresample   4.  4.100 /  4.  4.100
libpostproc    56.  4.100 / 56.  4.100
```