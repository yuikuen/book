# WireGuard Install(Transfer)

> 在现实情况中更多的场景是中转。

## 版本环境

WireGuard 是内核级别的特性，对内核版本是有要求的 Kernel > 3.10(`uname -r`)

- System：CentOS7.9.2009 Minimal
- Kernel：kernel-ml-5.18.14
- Client：Linux(CentOS)、Windows11

**实验场景**

以 VPS 作为中继服务，家庭网下的笔电访问到办公网下的各服务器设备

- 有固定公网 IP 的服务器(VPS)
- 办公网(office)、家庭网(home)均无固定 IP
- 办公网下搭建一台 Linux 服务器，用于分发内网路由

## 安装 WireGuard

> VPS & Office 都需要提前升级内核、开启内核转发功能

1）更新内核

```bash
# 可根据自行选择版本升级
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
$ rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
$ yum --enablerepo=elrepo-kernel install kernel-ml -y

# 查看可用内核并生成grub配置文件，再重新创建内核配置，重启生效
$ awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
$ grub2-set-default 0
$ grub2-mkconfig -o /boot/grub2/grub.cfg && init 6
```

2）更新源并下载安装 `WireGuard`

```bash
$ yum install epel-release https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
$ yum install yum-plugin-elrepo
$ yum install kmod-wireguard wireguard-tools
```

3）开启内核转发功能

```bash
$ echo 1 > /proc/sys/net/ipv4/ip_forward
$ echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
$ echo "net.ipv4.conf.all.proxy_arp = 1" >> /etc/sysctl.conf
$ sysctl -p
```

## Peer1 配置

> VPS 中转服务器，配置 `iptables` 转发规则，将 Peer3 的流量转发给 Peer2

1）创建各 Peer 密钥文件

```bash
$ mkdir -p /etc/wireguard/cert 
$ cd !$
$ wg genkey | tee privatekey-server | wg pubkey > publickey-server
$ wg genkey | tee office-privatekey | wg pubkey > office-publickey
$ wg genkey | tee home-privatekey | wg pubkey > home-publickey
```

2）创建配置文件

```bash
$ touch /etc/wireguard/wg0.conf; vim !$
[Interface]
Address = 6.6.6.1/24
SaveConfig = false
DNS = 8.8.8.8
PostUp = iptables -I FORWARD -i %i -j ACCEPT
PostUp = iptables -I FORWARD -o %i -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -D FORWARD -o %i -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820
PrivateKey = YLclRLq4RAlzTz4qz9+E4lBFmlJv9OUw43P7cQlNXnc= 

[peer]
# Office 
PublicKey = cfRUoERvMRp2IXsmzhYhChYMp+Zfaes0ROlmz8+4nlo= 
AllowedIPs = 6.6.6.2/24,188.188.3.33/24,188.188.4.0/24

[Peer]
# Home
PublicKey = 1YjZk/t/5HYiSMaYjnQurS4VwlCn0ip5MvY75iCg0Fs=
AllowedIPs = 6.6.6.3/24
```

3）启动服务并检测状态

```bash
$ systemctl enable --now wg-quick@wg0
$ wg
interface: wg0
  public key: EX7D+TfIMZprYsUCXH0hzYU4/ud//Zr8nF9iRX0hFQA=
  private key: (hidden)
  listening port: 51820

peer: cfRUoERvMRp2IXsmzhYhChYMp+Zfaes0ROlmz8+4nlo=
  endpoint: 183.xxx.xxx.xxx:44326
  allowed ips: 188.188.4.0/24
  latest handshake: 29 seconds ago
  transfer: 100.45 KiB received, 53.17 KiB sent

peer: 1YjZk/t/5HYiSMaYjnQurS4VwlCn0ip5MvY75iCg0Fs=
  endpoint: 14.xxx.xxx.xxx:47709
  allowed ips: 6.6.6.0/24
  latest handshake: 27 minutes, 50 seconds ago
  transfer: 60.13 KiB received, 37.18 KiB sent
```

## Peer2 配置

> 本地使用 `iptables` 进行内网转发，指定 Peer 为中转服务器 Peer1，并配置 endpoint

```bash
$ cat /etc/wireguard/wg0.conf
[Interface]
Address = 6.6.6.2/32
DNS = 114.114.114.114
PostUp = iptables -I FORWARD -i %i -j ACCEPT
PostUp = iptables -I FORWARD -o %i -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -D FORWARD -o %i -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PrivateKey = OJ+KNIEQBWkW+nsqj2QgxigKHum5EJ2PXMKwG8Z/Ylw=

[Peer]
# publickey-server公钥文件
PublicKey = EX7D+TfIMZprYsUCXH0hzYU4/ud//Zr8nF9iRX0hFQA= 
AllowedIPs = 6.6.6.0/24
Endpoint = 107.148.13.1:51820 
PersistentKeepalive = 15
```

3）启动服务并检测状态

```bash
$ systemctl enable --now wg-quick@wg0
$ wg
interface: wg0
  public key: cfRUoERvMRp2IXsmzhYhChYMp+Zfaes0ROlmz8+4nlo=
  private key: (hidden)
  listening port: 44326

peer: EX7D+TfIMZprYsUCXH0hzYU4/ud//Zr8nF9iRX0hFQA=
  endpoint: 107.xxx.xxx.xxx:51820
  allowed ips: 6.6.6.0/24
  latest handshake: 1 minute, 7 seconds ago
  transfer: 80.73 KiB received, 246.69 KiB sent
  persistent keepalive: every 15 seconds
```

## Peer3 配置

> Peer3 为 Windows 客户端，配置只作为流量接收，不做转发，只需配置 Peer1 的 endpoint

```bash
[Interface]
# GKUp9XLEsVdYQl1NMo4PCyOoJ7N0zxMrUXrvolwDaXk=
PrivateKey = GKUp9XLEsVdYQl1NMo4PCyOoJ7N0zxMrUXrvolwDaXk=
Address = 6.6.6.3/24

[Peer]
# publickey-server公钥文件
PublicKey = EX7D+TfIMZprYsUCXH0hzYU4/ud//Zr8nF9iRX0hFQA=
AllowedIPs = 6.6.6.0/24, 188.188.3.33/24, 188.188.4.0/24
Endpoint = 107.148.13.1:51820
PersistentKeepalive = 15
```

## 基本操作

```bash
# 不中断活跃连接的情况下重新加载配置文件：
$ wg syncconf wg0 <(wg-quick strip wg0)

$ wg-quick down wg0 && wg-quick up wg0
```

## 参考链接

- [被Linux创始人称做艺术品的组网神器——WireGuard](https://zhuanlan.zhihu.com/p/447375895)
- [CentOS7 下安装 WireGuard](https://www.cnblogs.com/zhenxing06/p/16049288.html)
- [Centos部署安装wireguard实现办公组网](https://www.wabks.com/post/1129.html#ecs)