# Ceph Install(Demo1)

## 版本环境

- System：CentOS7.9.2009 Minimal
- Ceph：13.2.10

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

> 场景：四台服务器，可上外网，其中三台作为 ceph 集群，一台作为 cehp 客户端，除了 client 外每台加一个磁盘（/dev/sdb），无需分区

## 安装程序

1）配置主机名及 hosts

```bash
$ cat >> /etc/hosts << EOF
188.188.4.110 client
188.188.4.111 ceph01
188.188.4.112 ceph02
188.188.4.113 ceph03
EOF
```

2）设置 ntpdate 时间同步并设置在 crontab 内

```bash
$ rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
$ yum install ntpdate -y
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
$ echo 'Asia/Shanghai' >/etc/timezone
$ ntpdate time2.aliyun.com
$ crontab -e
# 加入到crontab
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com
```

3）配置 Ceph-yum 源

```bash
$ yum install epel-release -y
$ cat > /etc/yum.repos.d/ceph.repo < EOF
[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/x86_64/
enabled=1
gpgcheck=0
priority=1
 
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/
enabled=1
gpgcheck=0
priority=1
 
[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
enabled=1
gpgcheck=0
priority=1
EOF
```

4）更新系统并配置 ssh 免密登录（以 ceph01 为部署节点，配置使 ceph01 可以免密登录四台主机）

```bash
$ yum -y update
$ ssh-keygen -t rsa
$ ssh-copy-id ceph01
$ ssh-copy-id ceph02
$ ssh-copy-id ceph03
$ ssh-copy-id client
```

在 ceph01 上ssh其他主机无需输入密码代表配置 ssh 成功

## 安装工具

1）在 ceph01 上安装部署工具（其他节点不用安装）

```bash
$ yum install ceph-deploy -y
```

2）每个节点都建立一个集群配置目录

```bash
$ mkdir /etc/ceph
$ cd /etc/ceph
```

## 创建集群

```bash
$ ceph-deploy new ceph01
Traceback (most recent call last):
  File "/usr/bin/ceph-deploy", line 18, in <module>
    from ceph_deploy.cli import main
  File "/usr/lib/python2.7/site-packages/ceph_deploy/cli.py", line 1, in <module>
    import pkg_resources
ImportError: No module named pkg_resources
# 报错处理，缺少python依赖
$ yum install python-setuptools
```

```bash
# 成功执行过程如下：
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy new ceph01
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  func                          : <function new at 0x7f9f8a0a1de8>
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f9f8981b518>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  ssh_copykey                   : True
[ceph_deploy.cli][INFO  ]  mon                           : ['ceph01']
[ceph_deploy.cli][INFO  ]  public_network                : None
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  cluster_network               : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  fsid                          : None
[ceph_deploy.new][DEBUG ] Creating new cluster named ceph                             # 创建新的集群 ceph
[ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds                       # ssh 免密登录 ceph01 成功
[ceph01][DEBUG ] connected to host: ceph01 
[ceph01][DEBUG ] detect platform information from remote host
[ceph01][DEBUG ] detect machine type
[ceph01][DEBUG ] find the location of an executable
[ceph01][INFO  ] Running command: /usr/sbin/ip link show
[ceph01][INFO  ] Running command: /usr/sbin/ip addr show
[ceph01][DEBUG ] IP addresses found: [u'188.188.4.111']
[ceph_deploy.new][DEBUG ] Resolving host ceph01                                       # 连接ceph01获取IP为188.188.4.111
[ceph_deploy.new][DEBUG ] Monitor ceph01 at 188.188.4.111
[ceph_deploy.new][DEBUG ] Monitor initial members are ['ceph01']
[ceph_deploy.new][DEBUG ] Monitor addrs are ['188.188.4.111']
[ceph_deploy.new][DEBUG ] Creating a random mon key...                                # 创建mon的key
[ceph_deploy.new][DEBUG ] Writing monitor keyring to ceph.mon.keyring...
[ceph_deploy.new][DEBUG ] Writing initial config to ceph.conf...                      # 创建ceph.conf配置文件
```

会在当前配置文件目录生成以下配置文件

```bash
ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring
```

说明：

- ceph.conf                        集群配置文件
- ceph-deploy-ceph.log   使用ceph-deploy部署的日志记录
- ceph.mon.keyring          mon的验证key文件 监控需要的令牌

## 节点加入

```bash
# 所有节点都需要安装
$ yum install ceph ceph-radosgw -y
$ ceph -v
ceph version 13.2.10 (564bdc4ae87418a232fc901524470e1a0f76d641) mimic (stable)
```

> 注意： 如公网OK,并且网速好的话,可以用`ceph-deploy install node1 node2 node3`命令来安装,但网速不好的话会比较坑，所以这里选择直接用准备好的本地 ceph 源,然后`yum install ceph ceph-radosgw -y`安装即可。

## 创建 mon 监控

> 一般执行 `ceph-deploy mon create-initial` 即可，但执行报错，则修改配置 `public network = 188.188.4.0/24` 再执行命令

1）解决命令执行错报错问题，修改 ceph01 配置文件在 `[global]` 配置端添加下面一句

```bash
$ vim ceph.conf 
public network = 188.188.4.0/24
```

2）监控节点初始化，并同步配置到所有节点(ceph01,ceph02,ceph03,不包括 client）

```bash
$ ceph-deploy mon create-initial
[node1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.mon][ERROR ] RuntimeError: config file /etc/ceph/ceph.conf exists with different content; use --overwrite-conf to overwrite
[ceph_deploy][ERROR ] GenericError: Failed to create 1 monitors
```

3）查看监控状态

```bash
$ ceph health
HEALTH_OK
```

4）将配置信息同步到所有节点

```bash
$ ceph-deploy admin ceph01 ceph02 ceph03
# 查看其他节点是否多了几个配置文件
$ ll
总用量 12
-rw------- 1 root root 151 8月   3 14:46 ceph.client.admin.keyring
-rw-r--r-- 1 root root 322 8月   3 14:46 ceph.conf
-rw-r--r-- 1 root root  92 4月  24 01:07 rbdmap
-rw------- 1 root root   0 8月   1 16:46 tmpDX_w1Z
```

5）再次查看状态

```bash
$ ceph -s
  cluster:
    id:     a9aa277d-7192-4687-b384-9787b73ece71
    health: HEALTH_OK                     # 集群监控状态OK
  
  services:
    mon: 1 daemons, quorum node1 (age 3m) # 1个mon
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in
  
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:    
```

**为了防止 mon 单点故障，你可以加多个 mon 节点(建议奇数个，因为有 quorum 仲裁投票）**

```bash
$ ceph-deploy mon add ceph02
$ ceph-deploy mon add ceph03
$ ceph -s
  cluster:
    id:     a9aa277d-7192-4687-b384-9787b73ece71
    health: HEALTH_OK
  
  services:
    mon: 3 daemons, quorum node1,node2,node3 (age 0.188542s) # 3个mon了
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in
  
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:    
```

**注：监控异常问题**，ceph 集群对时间同步要求非常高, 即使你已经将 ntpd 服务开启,但仍然可能有 `clock skew deteted` 相关警告

监控到时间不同步的解决方法：

1）**在 ceph 集群所有节点上(`node1`,`node2`,`node3`)操作，**不使用 ntpd 服务，直接使用 crontab 同步

```bash
$ systemctl stop ntpd
$ systemctl disable ntpd
$ crontab -e
*/10 * * * * ntpdate ntp1.aliyun.com  # 每5或10分钟同步1次公网的任意时间服务器
```

2）调大时间警告的阈值

```bash
$ vim /etc/ceph/ceph.conf
[global]                          # 在global参数组里添加以下两行                        
mon clock drift allowed = 2       # monitor间的时钟滴答数(默认0.5秒)
mon clock drift warn backoff = 30 # 调大时钟允许的偏移量(默认为5)
```

<font color=red>每次修改了 ceph.conf 配置需要加 `--overwrite-conf` 参数覆盖同步到所有节点，再所有 ceph 集群节点上重启 ceph-mon.target 服务</font>

```bash
$ ceph-deploy --overwrite-conf admin node1 node2 node3
$ systemctl restart ceph-mon.target
```

## 创建 mgr 管理

eph luminous 版本中新增加了一个组件：Ceph Manager Daemon，简称 ceph-mgr；

该组件的主要作用是分担和扩展 monitor 的部分功能，减轻 monitor 的负担，让更好地管理 ceph 存储系统；

```bash
$ ceph-deploy mgr create ceph01
```

添加多个 mgr 可以实现 HA

```bash
$ ceph-deploy mgr create ceph02
$ ceph-deploy mgr create ceph03
$ ceph -s
  cluster:
    id:     a9aa277d-7192-4687-b384-9787b73ece71
    health: HEALTH_WARN
            Module 'restful' has failed dependency: No module named 'pecan'
            Reduced data availability: 1 pg inactive
            OSD count 0 < osd_pool_default_size 3
  
  services:
    mon: 3 daemons, quorum node1,node2,node3 (age 11m)
    mgr: node1(active, since 3m), standbys: node2, node3  # 3个mgr其中node1为active node2 3为standbys
    osd: 0 osds: 0 up, 0 in
  
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     100.000% pgs unknown
             1 unknown
```

## 创建 osd 存储

```bash
# 查看相关帮忙,lsblk 查看已添加的磁盘情况
$ ceph-deploy disk --help
$ ceph-deploy osd --help
```

zap 表示干掉磁盘上的数据,相当于格式化

```bash
$ ceph-deploy disk zap ceph01 /dev/sdb
$ ceph-deploy disk zap ceph02 /dev/sdb
$ ceph-deploy disk zap ceph03 /dev/sdb
```

将磁盘创建为 osd

```bash
$ ceph-deploy osd create --data /dev/sdb ceph01
$ ceph-deploy osd create --data /dev/sdb ceph02
$ ceph-deploy osd create --data /dev/sdb ceph03
$ ceph -s
  cluster:
    id:     a9aa277d-7192-4687-b384-9787b73ece71
    health: HEALTH_WARN
            Module 'restful' has failed dependency: No module named 'pecan'
  
  services:
    mon: 3 daemons, quorum ceph01,ceph02,ceph03 (age 22m)
    mgr: ceph01(active, since 14m), standbys: ceph02, ceph03
    osd: 3 osds: 3 up (since 4s), 3 in (since 4s) # 3个osd 3个up启用状态 3 in代表3G在使用 每块硬盘占用1G
  
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 45 GiB / 48 GiB avail  # 3G使用 45G空闲 总计48G及三块16G硬盘相加之和
    pgs:     1 active+clean
```

## 集群扩容

假设再加一个新的集群节点 ceph04

1. 主机名配置和绑定
2. 在 ceph04 上 `yum install ceph ceph-radosgw -y` 安装软件
3. 在部署节点 ceph01 上同步配置文件给 ceph04，`ceph-deploy admin ceph04`
4. 按需求选择在 ceph04 上添加 mon 或 mgr 或 osd 等

## Ceph-Dashboard

1）查看集群状态确认 mgr 的 active 节点

```bash
$ ceph -s
  cluster:
    id:     134c6247-2fd4-4669-8c19-d0c9200d90c8
    health: HEALTH_OK
  
  services:
    mon: 3 daemons, quorum ceph01,ceph02,ceph03
    mgr: ceph01(active), standbys: ceph02, ceph03  # 确认mgr active节点为 ceph01
    osd: 3 osds: 3 up, 3 in
    rgw: 1 daemon active
  
  data:
    pools:   4 pools, 32 pgs
    objects: 189  objects, 1.5 KiB
    usage:   3.0 GiB used, 45 GiB / 48 GiB avail
    pgs:     32 active+clean
```

2）查看开启及关闭的模块

```bash
$ ceph mgr module ls
{
    "enabled_modules": [
        "balancer",
        "crash",
        "dashboard",
        "iostat",
        "restful",
        "status"
    ],
    "disabled_modules": [
        {
            "name": "hello",
            "can_run": true,
            "error_string": ""
        },
        {
            "name": "influx",
            "can_run": false,
            "error_string": "influxdb python module not found"
        },
        {
            "name": "localpool",
            "can_run": true,
            "error_string": ""
        },
        {
            "name": "prometheus",
            "can_run": true,
            "error_string": ""
        },
        {
            "name": "selftest",
            "can_run": true,
            "error_string": ""
        },
        {
            "name": "smart",
            "can_run": true,
            "error_string": ""
        },
        {
            "name": "telegraf",
            "can_run": true,
            "error_string": ""
        },
        {
            "name": "telemetry",
            "can_run": true,
            "error_string": ""
        },
        {
            "name": "zabbix",
            "can_run": true,
            "error_string": ""
        }
    ]
}
```

3）**开启 dashboard 模块**

```bash
$ ceph mgr module enable dashboard
```

如开启报错，可按下述处理

```bash
$ ceph mgr module enable dashboard
Error ENOENT: all mgr daemons do not support module 'dashboard', pass --force to force enablement

# 需要在每个开启 mgr 的节点安装 ceph-mgr-dashboard
$ yum install ceph-mgr-dashboard -y
```

注意：不能仅仅在 active 节点安装，需要在 standby 节点都安装

4）**创建自签名证书**

```bash
$ ceph dashboard create-self-signed-cert
Self-signed certificate created

# 生成密钥对，并配置给ceph mgr
$ mkdir /etc/mgr-dashboard
$ cd /etc/mgr-dashboard/
$ openssl req -new -nodes -x509 -subj "/O=IT-ceph/CN=cn" -days 3650 -keyout dashboard.key -out dashboard.crt -extensions v3_ca
dashboard.crt  dashboard.key
```

5）**配置 mgr services**

在 ceph 集群的 active mgr 节点上配置 mgr services

```bash
$ ceph config set mgr mgr/dashboard/server_addr 188.188.4.111
$ ceph config set mgr mgr/dashboard/server_port 8080
```

重启 dashboard 模块,并查看访问地址(注意：不重启查看的端口是默认的8443端口，无法访问)

6）**重启 dashboard 模块**

```bash
$ ceph mgr module disable dashboard
$ ceph mgr module enable dashboard
# 查看mgr service
$ ceph mgr services
{
    "dashboard": "https://192.168.1.101:8080/"
}
```

7）设置访问 web 页面用户名和密码

```bash
$ ceph dashboard set-login-credentials admin admin
```

通过本机或其它主机访问 https://ip:8080

## Ceph 文件存储

要运行 Ceph 文件系统，必须先装只是带一个 mds 的 Ceph 存储集群

Ceph.MDS：为 Ceph 文件存储类型存放元数据 metadata（也就是说 Ceph 块存储和 Ceph 对象存储不使用 MDS）

1）增设配置：在 ceph01 部署节点上修改配置 /etc/ceph/ceph.conf  增加配置

```bash
mon_allow_pool_delete = true
```

2）同步配置文件，注意修改了配置文件才需要同步，没有修改不需要同步配置文件

```bash
$ ceph-deploy --overwrite-conf admin ceph01 ceph02 ceph03
```

3）创建 mds

```bash
$ ceph-deploy mds create ceph01 ceph02 ceph03
```

一个 Ceph 文件系统需要至少两个 RADOS 存储池，一个用于数据，一个用于元数据

```bash
$ ceph osd pool create cephfs_pool 128
pool 'cephfs_pool' created
$ ceph osd pool create cephfs_metadata 64
pool 'cephfs_metadata' created
```

参数解释
- 创建 pool 自定义名为 cephfs_pool 用于存储数据 PG 数为 128 (因用于存储数据所以 PG 数较大)
- 创建 pool 自定义名为 cephfs_metadata 用于存储元数据 PG 数为 64

- 少于5个OSD则PG数为128，5-10个OSD则PG数为512，10-50个OSD则PG数为1024，更多的OSD需要自行计算

4）查看创建的 pool 详细信息

```bash
$ ceph osd pool ls |grep cephfs
cephfs_pool
cephfs_metadata

$ ceph osd pool get cephfs_pool all
size: 3
min_size: 2
pg_num: 128
pgp_num: 128
crush_rule: replicated_rule
hashpspool: true
nodelete: false
nopgchange: false
nosizechange: false
write_fadvise_dontneed: false
noscrub: false
nodeep-scrub: false
use_gmt_hitset: 1
fast_read: 0
pg_autoscale_mode: warn
```

6）创建文件系统

创建 Ceph 文件系统,并确认客户端访问的节点

```bash
$ ceph fs new cephfs cephfs_metadata cephfs_pool
new fs with metadata pool 10 and data pool 9
```
7）查看状态

```bash
$ ceph osd pool ls
.rgw.root
default.rgw.control
default.rgw.meta
default.rgw.log
cephfs_pool
cephfs_metadata

$ ceph fs ls   # 确认metadata池和data池        
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_pool ]

$ ceph mds stat
...
cephfs-1/1/1 up  {0=ceph03=up:active}, 2 up:standby # ceph03为active,up状态;其他为standby
```

注：metadata 保存在 ceph03 上

8）Key 文件生成

客户端准备验证 key 文件，ceph 默认启用了 cephx 认证, 所以客户端的挂载必须要验证（ceph.conf 默认配置文件开启）

可在集群节点(ceph01,ceph02,ceph03)上任意一台查看密钥字符串

```bash
$ cat /etc/ceph/ceph.client.admin.keyring
$ ceph-authtool -p /etc/ceph/ceph.client.admin.keyring >admin.key
```

把这个文件放在客户端`client /root/admin.key`(注意：直接把 key 复制编辑 admin.key 文档可能会在挂载时报错)

9）客户端挂载

```bash
$ yum -y install ceph-fuse
$ mount -t ceph ceph01:6789:/ /mnt -o name=admin,secretfile=/root/admin.key
$ df -h
```

需要安装否则客户端不支持，也可以使用其他 node 主机名进行挂载，例如 ceph03

注意：如使用文件挂载报错，可以使用参数 secret=秘钥 进行挂载；也可以使用两个客户端，同时挂载此文件存储，实现同读同写；

10）验证集群

往挂载的硬盘写数据可以在 dashboard 查看读写监控状态

```bash
$ dd if=/dev/zero of=/mnt/file1 bs=10M count=1000
```

11）删除文件存储

在所有挂载了文件存储的客户端卸载文件挂载，并停掉所有节点的 mds

```bash
$ umount /mnt
$ systemctl stop ceph-mds.target
```

回到集群任意一个节点上(node1,node2,node3 其中之一)删除

**如果要客户端删除,需要在 node1 上 `ceph-deploy admin client` 同步配置才可以**

```bash
$ ceph fs rm cephfs --yes-i-really-mean-it
$ ceph osd pool delete cephfs_metadata cephfs_metadata --yes-i-really-really-mean-it
pool 'cephfs_metadata' removed
$ ceph osd pool delete cephfs_pool cephfs_pool --yes-i-really-really-mean-it
pool 'cephfs_pool' removed
```

注意：为了安全需要输入两次创建的pool名并且加参数 `--yes-i-really-really-mean-it` 才能删除

另注意：需要在配置文件添加以下配置，才能删除

```bash
mon_allow_pool_delete = true
```

如还提示报错：

```bash
Error EPERM: pool deletion is disabled; you must first set the mon_allow_pool_delete config option to true before you can destroy a pool
```

则重启服务 ceph-mon.target 即可`$ systemctl restart ceph-mon.target`

上述执行后，再重新开启所有节点的 mds 服务

```bash
$ systemctl start ceph-mds.target
```

## Ceph 块存储

### 创建

1）同步配置：在 ceph01上同步配置文件到 client

```bash
$ cd /etc/ceph
$ ceph-deploy admin client
```

2）建议存储池，并初始化（在客户端 client 操作）

```bash
$ ceph osd pool create rbd_pool 128
pool 'rbd_pool' created

$ rbd pool init rbd_pool
```

3）创建存储卷，一个存储卷（这里卷名为 volume1 大小为 5000M）

```bash
$ rbd create volume1 --pool rbd_pool --size 5000

# 查看创建状态
$ rbd ls rbd_pool
volume1
$ rbd info volume1 -p rbd_pool
rbd image 'volume1':         # volume1 为 rbd_image
    size 4.9 GiB in 1250 objects
    order 22 (4 MiB objects)
    id: 60fc6b8b4567
    block_name_prefix: rbd_data.60fc6b8b4567
    format: 2                # 两种格式1和2 默认是2
    features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
    op_features:
    flags:
    create_timestamp: Tue Aug  4 09:20:05 2020
```

4）映射块设备，将创建的卷映射成块设备，因为 rbd 镜像的一些特性，OS kernel 并不支持，所以映射报错

```bash
$ rbd map rbd_pool/volume1
rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable rbd_pool/volume1 object-map fast-diff deep-flatten".
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (6) No such device or address
```

**解决办法：disable 掉相关特性**

```bash
$ rbd feature disable rbd_pool/volume1 exclusive-lock object-map fast-diff deep-flatten
```

再次映射

```bash
$ rbd map rbd_pool/volume1
/dev/rbd0

$ rbd showmapped
id pool     image   snap device   
0  rbd_pool volume1 -    /dev/rbd0
```

创建了磁盘/dev/rbd0 类似于做了一个软连接

**如果需要取消映射可以使用命令**

```bash
$ rbd unmap /dev/rbd0
```

5）格式化挂载

```bash
$ mkfs.xfs /dev/rbd0
$ mount /dev/rbd0 /mnt
$ df -h|tail -1
/dev/rbd0                4.9G   33M  4.9G    1% /mnt
```

### 扩容

1）如扩容成8000M

```bash
$ rbd resize --size 8000 rbd_pool/volume1
Resizing image: 100% complete...done.

# 查看并没有变化
$ df -h|tail -1
/dev/rbd0                4.9G   33M  4.9G    1% /mnt
```

2）动态刷新扩容

```bash
$ xfs_growfs -d /mnt/
meta-data=/dev/rbd0              isize=512    agcount=8, agsize=160768 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=1280000, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1280000 to 2048000

# 再次查看，扩容成功
$ df -h|tail -1
/dev/rbd0                7.9G   33M  7.8G    1% /mnt
```

注意：该命令和 LVM 扩容命令一致

### 缩容

1）**块存储裁减**不能在线裁减，裁减后需要重新格式化再挂载，如果有数据需要提前备份好数据，如裁减为5000M

```bash
$ rbd resize --size 5000 rbd_pool/volume1 --allow-shrink
Resizing image: 100% complete...done.
```

2）卸载，格式化

```bash
$ umount /mnt
$ fdisk -l
$ mkfs.xfs -f /dev/rbd0
$ mount /dev/rbd0 /mnt/

$ df -h|tail -1
/dev/rbd0                4.9G   33M  4.9G    1% /mnt
```

### 删除

```bash
#卸载
$ umount /mnt
#取消映射
$ rbd unmap /dev/rbd0
#删除存储池
$ ceph osd pool delete rbd_pool rbd_pool --yes-i-really-really-mean-it
pool 'rbd_pool' removed
```

## Ceph 对象存储

1）创建 rgw（在 ceph01 上创建 rgw）

```bash
$ ceph-deploy rgw create ceph01

# 查看,运行端口是7480
$ lsof -i:7480
```

2）测试连接，在客户端测试连接对象网关，在 client 安装测试工具，创建一个测试用户，需要在部署节点使用 `ceph-deploy admin client` 同步配置文件给 client

```bash
$ radosgw-admin user create --uid="testuser" --display-name="First User"|grep -E 'access_key|secret_key'
            "access_key": "S859J38AS6WW1CZSB90M",
            "secret_key": "PCmfHoAsHw2GIEioSWvN887o02VXesOkX2gJ20fG"
```

```bash
$ radosgw-admin user create --uid="testuser" --display-name="First User"
{
    "user_id": "testuser",
    "display_name": "First User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "testuser",
            "access_key": "S859J38AS6WW1CZSB90M",
            "secret_key": "PCmfHoAsHw2GIEioSWvN887o02VXesOkX2gJ20fG"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```

上面一大段主要有用的为 access_key 与 secret_key，用于连接对象存储网关

3）S3连接对象网关，客户端安装 s3cmd 工具，并编写配置文件

```bash
$ yum install s3cmd
```

4）创建配置文件，内容如下

```bash
$ cat /root/.s3cfg
[default]
access_key = S859J38AS6WW1CZSB90M
secret_key = PCmfHoAsHw2GIEioSWvN887o02VXesOkX2gJ20fG
host_base = 192.168.1.101:7480
host_bucket = 192.168.1.101:7480/%(bucket)
cloudfront_host = 192.168.1.101:7480
use_https = False
```

5）列出 bucket

```bash
$ s3cmd ls
```

6）创建一个桶，上传、下载测试

```bash
$ s3cmd mb s3://test_bucket              # 创建
$ s3cmd put /etc/fstab s3://test_bucket  # 上传
$ s3cmd get  s3://test_bucket/fstab      # 下载
```


