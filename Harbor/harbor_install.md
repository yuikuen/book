# Harbor Install(Offline)

> 本文主要参照 [官方文档](https://goharbor.io/docs/2.5.0/install-config/configure-https/) 进行离线部署 Harbor 私有仓库


## 版本环境

- System：CentOS7.9.2009 Minimal
- Runtime：docker-ce-20.10.*
- Docker-compose：
- Harbor：

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## Docker

> [Docker](https://docs.docker.com/engine/install/centos/) 要求 CentOS 系统的内核版本高于 3.10，安装前提前升级内核

1）安装需要的软件包，`yum-util` 提供 `yum-config-manager` 功能，另外两个是 `devicemapper` 驱动依赖

```bash
$ yum -y install yum-utils device-mapper-persistent-data lvm2
```

2）设置存储库

```bash
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

由于国内访问不到 Docker 官方镜像，可以通过 aliyun 的源来完成

```bash
$ yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
$ yum makecache fast
```

3）安装程序，默认最新版(**docker-ce:社区版 ee:企业版**)

```bash
$ yum list docker-ce --showduplicates | sort -r
```

安装指定版本的 Docker-CE: (VERSION 例如上面的 3:20.10.17-3.el7)

```bash
$ yum -y install docker-ce-[VERSION]
$ yum -y install docker-ce-20.10.* docker-cli-20.10.*

$ docker version
Client: Docker Engine - Community
 Version:           20.10.17
 API version:       1.41
 Go version:        go1.17.11
 Git commit:        100c701
 Built:             Mon Jun  6 23:05:12 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

4）设置防火墙规则，让其通行(可略，根据实际环境操作)

```bash
$ vim /lib/systemd/system/docker.service
[Service]
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
```

5）启动服务并配置加速器

```bash
$ systemctl enable --now docker
```

> 镜像加速器：阿里云加速器，daocloud 加速器，中科大加速器，Docker 中国官方镜像等

```bash
$ cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": [
      "https://eihzr0te.mirror.aliyuncs.com",
      "https://dockerhub.mirrors.nwafu.edu.cn/",
      "https://mirror.ccs.tencentyun.com",
      "https://docker.mirrors.ustc.edu.cn/",
      "https://reg-mirror.qiniu.com",
      "http://hub-mirror.c.163.com/",
      "https://registry.docker-cn.com"]
}
EOF

$ systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```
注：修改配置后需要重启相关服务

## Docker-Compose

> [Docker Compose](https://docs.docker.com/compose/install/) 是 docker 提供的一个命令行工具，用来定义和运行由多个容器组成的应用

不建议采用官网下载方式(国内访问较慢)，直接在 [GitHub](https://github.com/docker/compose/releases) 选择版本进行下载即可

```bash
$ wget https://github.com/docker/compose/releases/download/v2.10.0/docker-compose-linux-x86_64
$ mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
$ docker-compose --version
Docker Compose version v2.10.0
```

## Harbor

> 项目需求：配置 harbor 支持域名以 https 方式对外提供服务

一般公司企业都会有所属的证书，而这里只是测试环境，所以按官方操作指引，使用自签证书

1）创建程序目录并到 [GitHub](https://github.com/goharbor/harbor/releases) 下载离线包

```bash
$ mkdir -p /opt/harbor/{software,cert}
$ cd /opt/harbor/software
$ https://github.com/goharbor/harbor/releases/download/v2.4.3/harbor-offline-installer-v2.4.3.tgz
```

### 配置证书

1）生成证书颁发机构的证书

```bash
$ cd /opt/harbor/cert
$ openssl genrsa -out ca.key 4096

# 调整`-subj`选项中的值以反映您的组织。如果使用FQDN连接Harbor主机，则必须将其指定为公用名 ( `CN`) 属性.
$ openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=DongGuan/L=DongGuan/O=example/OU=Personal/CN=harbor.yuikuen.top" \
 -key ca.key \
 -out ca.crt
```

2）生成服务器证书

```bash
# 生成私钥并生成证书签名请求
$ openssl genrsa -out harbor.yuikuen.top.key 4096
$ openssl req -sha512 -new \
    -subj "/C=CN/ST=DongGuan/L=DongGuan/O=example/OU=Personal/CN=harbor.yuikuen.top" \
    -key harbor.yuikuen.top.key \
    -out harbor.yuikuen.top.csr

# 生成x509 v3扩展文件
# 无论是使用 FQDN 还是 IP 地址连接到 Harbor 主机，都必须创建此文件，以便为 Harbor 主机生成符合主题备用名称 (SAN) 和 x509 v3 的证书扩展要求
$ cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.yuikuen.top
DNS.2=harbor.yuikuen
DNS.3=harbor
EOF

# 使用v3.ext文件为Harbor主机生成证书，将CRS和CRT文件名替换为Harbor主机名
$ openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.yuikuen.top.csr \
    -out harbor.yuikuen.top.crt
```

3）提供证书，将生成后的 `ca.crt`，`yourdomain.com.crt` 和 `yourdomain.com.key` 文件提供给 Harbor 和 Docker

> Docker 需要将 CRT 文件作为 CA 证书，CERT 文件作为客户端证书

```bash
$ openssl x509 -inform PEM -in harbor.yuikuen.top.crt -out harbor.yuikuen.top.cert

# 将服务器证书、密钥和 CA 文件复制到 Harbor 主机上的 Docker 证书文件夹中
$ mkdir -p /etc/docker/certs.d/harbor.yuikuen.top
$ cp ca.crt *.top.cert *.top.key /etc/docker/certs.d/harbor.yuikuen.top/
```

**如果将默认 `nginx` 端口 443映射到其他端口，需要按下述格式创建文件夹**

`/etc/docker/certs.d/yourdomain.com:port` 或 `/etc/docker/certs.d/harbor_IP:port`

4）重启服务

```bash
$ systemctl daemon-reload
$ systemctl restart docker
```

**证书配置说明：**

```bash
$ tree /etc/docker/certs.d/
/etc/docker/certs.d/
└── harbor.yuikuen.top
    ├── ca.crt                   <-- 签署注册表证书的证书颁发机构
    ├── harbor.yuikuen.top.cert  <-- 由CA签名的服务器证书
    └── harbor.yuikuen.top.key   <-- 由CA签名的服务器密钥

1 directory, 3 files
```

### 程序部署

1）解压后直接修改配置文件，若采用 http 方式，前述配置可忽略并将其配置注释

```bash
$ tar -xf harbor-offline-installer-v2.4.1.tgz -C /opt
$ cp harbor.yml.tmpl harbor.yml && vim harbor.yml
```

```yaml
# Configuration file of Harbor <--修改域名地址
hostname: harbor.yuikuen.top

# http related config
http:
  port: 80
  
# https related config <--添加证书
https:
  port: 443
  certificate: /opt/harbor/cert/harbor.yuikuen.top.cert
  private_key: /opt/harbor/cert/harbor.yuikuen.top.key

harbor_admin_password: Harbor12345

# Harbor DB configuration
database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900

# The default data volume
data_volume: /data
```

2）执行预备脚本`./prepare` ，待测试完出现 `Successfully called func: create_root_cert` 表示可正常部署

```bash
$ ./install.sh --with-chartmuseum --with-notary --with-trivy
```

- `--with-chartmuseum` 是安装chart仓库，不使用helm可不添加该参数
- `--with-notary` 含义是启用镜像签名，必须是 https 才可以，否则会报错`ERROR:root:Error: the protocol must be https when Harbor is deployed with Notary`


3）配置开机自动启服务

```bash
$ cat /usr/lib/systemd/system/harbor.service
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/vmware/harbor

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f  /opt/harbor/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /opt/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target
```

## 验证功能

为 Harbor 设置 HTTPS 后，可以通过执行以下步骤来验证 HTTPS 连接

- 打开浏览器并输入[https://yourdomain.com](https://yourdomain.com/)。它应该显示 Harbor 界面。

  某些浏览器可能会显示一条警告，指出证书颁发机构 (CA) 未知。使用不是来自受信任的第三方 CA 的自签名 CA 时会发生这种情况。您可以将 CA 导入浏览器以消除警告。

- 在运行 docker 守护进程的计算机上，检查`/etc/docker/daemon.json`文件，确保没有为https://yourdomain.com.设置`-insecure-Registry`选项。

- 从 Docker 客户端登录 Harbor `docker login yourdomain.com`

​       如果已将`nginx`443 端口映射到其他端口，请在`login`命令中添加该端口 `docker login yourdomain.com:port`

- 如果安装失败，请参阅 [Harbor 安装故障排除](https://goharbor.io/docs/2.0.0/install-config/troubleshoot-installation/)

**疑难解答**

如果使用来自证书颁发者的中间证书，请将中间证书与您自己的证书合并以创建证书包。运行以下命令

```bash
$ cat intermediate-certificate.pem >> yourdomain.com.crt
```

当 Docker 守护程序在某些操作系统上运行时，可能需要在操作系统级别信任证书。例如，运行以下命令

```bash
# Ubuntu
$ cp yourdomain.com.crt /usr/local/share/ca-certificates/yourdomain.com.crt 
$ update-ca-certificates

# CentOS
$ cp yourdomain.com.crt /etc/pki/ca-trust/source/anchors/yourdomain.com.crt
$ update-ca-trust
```