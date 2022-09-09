# NFS Install(YUM)

> CentOS7.x 简单部署 NFS 服务器

## 版本环境

- System：CentOS7.9.2009 Minimal
- NFS：nfs-utils rpcbind

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 服务端

1）配置环境

```bash
# 停止并禁用防火墙
$ systemctl stop firewalld
$ systemctl disable firewalld

# 关闭并禁用 SELinux
$ setenforce 0
$ sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

2）安装服务

```bash
$ yum install -y nfs-utils rpcbind

# 创建文件夹
$ mkdir /nfs

# 更改归属组与用户
$ chown -R nfsnobody:nfsnobody /nfs
```

3）配置 NFS

```bash
# 编辑exports
$ vi /etc/exports

# 输入以下内容(格式：FS共享的目录 NFS客户端地址1(参数1,参数2,...) 客户端地址2(参数1,参数2,...))
$ /nfs 188.188.4.0/24(rw,async,no_root_squash)
```

> 如果设置为 /nfs *(rw,async,no_root_squash) 则对所以的IP都有效

- 常用选项：
  - ro：客户端挂载后，其权限为只读，默认选项；
  - rw:读写权限；
  - sync：同时将数据写入到内存与硬盘中；
  - async：异步，优先将数据保存到内存，然后再写入硬盘；
  - Secure：要求请求源的端口小于1024
- 用户映射：
  - root_squash:当NFS客户端使用root用户访问时，映射到NFS服务器的匿名用户；
  - no_root_squash:当NFS客户端使用root用户访问时，映射到NFS服务器的root用户；
  - all_squash:全部用户都映射为服务器端的匿名用户；
  - anonuid=UID：将客户端登录用户映射为此处指定的用户uid；
  - anongid=GID：将客户端登录用户映射为此处指定的用户gid

4）设置自启动

```bash
$ systemctl restart rpcbind
$ systemctl enable nfs && systemctl restart nfs

# 查看是否有可用的NFS地址
$ showmount -e 127.0.0.1
```

## 客户端

1）安装服务

```bash
$ yum install -y nfs-utils rpcbind

# 创建挂载的文件夹，并挂载 nfs
$ mkdir -p /nfs-data
$ mount -t nfs -o nolock,vers=4 188.188.4.161:/nfs /nfs-data
$ systemctl enable nfs && systemctl restart nfs

# 设置开机自挂载
vim /etc/rc.d/rc.local
#在文件最后添加一行：
mount -t nfs -o nolock,vers=4 188.188.4.161:/nfs /nfs-data
```

- 参数解释：
  - mount：挂载命令
  - -o：挂载选项
  - nfs :使用的协议
  - nolock :不阻塞
  - vers : 使用的NFS版本号
  - IP : NFS服务器的IP（NFS服务器运行在哪个系统上，就是哪个系统的IP）
  - /nfs: 要挂载的目录（Ubuntu的目录）
  - /nfs-data : 要挂载到的目录（开发板上的目录，注意挂载成功后，/mnt下原有数据将会被隐藏，无法找到）

2）查看状态

```bash
$ df -h

# 查看nfs服务端信息
$ nfsstat -s

# 查看nfs客户端信息
$ nfsstat -c
```

- 卸载挂载

```bash
$ umount /nfs-data
```
