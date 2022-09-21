# DevOps Demon

## 1. 基础环境

### 1.1 Esxi 安装

> 环境都是在私有 Esxi 服务器上操作，如按教程搭建不成功，请检查自身环境或程序版本之类

…待更新

### 1.2 项目规划

> 内网服务器搭建了 DNS 服务，需要自行 hosts 文件或修改网络连接的 DNS 配置
>
> 配置方案仅供参考，可根据实际资源自行合并或拆分部署

| 角色              | 配置                        | 主机名  | IP地址        | 服务                        |
| ----------------- | --------------------------- | ------- | ------------- | --------------------------- |
| Esxi              | 36C/128G/5.11 TB(M2+SSD+HD) | devops  | 188.188.3.33  | 核心服务器                  |
| Client            | 4C/8G/60G                   | debug   | 188.188.4.44  | 客户端&测试机               |
| Harbor、SonarQube | 4C/4G/200G                  | harbor  | 188.188.4.220 | 镜像仓库/代码审查           |
| Gitlab            | 4C/4G/100G                  | gitlab  | 188.188.4.221 | 代码仓库                    |
| Jenkins           | 4C/4G/100G                  | jenkins | 188.188.4.222 | 发布平台                    |
| NextCloud         | 4C/4G/60G+1.5T              | cloud   | 188.188.4.244 | 网盘、文件服务器、DNS服务器 |

…待更新

### 1.2 系统安装

> 默认所有主机安装方式一致，仅供参考。

#### 1.2.1 CentOS 安装

1. 登录 Esxi 后点击 `主机` 或 `虚拟机`，然后点击菜单 `创建/注册虚拟机`

![1001](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1001.png)

2. 创建类型选择 `创建新虚拟机`

![1002](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1002.png)

3. 此外名称可自定义，而其它根据自身 ISO 进行选择操作系统系列&版本，兼容性默认即可

![1003](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1003.png)

4. 选择存储即为创建后的虚拟机存储目录，类型根据自身资源选择

![1004](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1004.png)

5. 自定义设置：主要配置为CPU、内存、硬盘、网络、光盘引导

![1005](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1005.png)

配置CPU、内存建议开启热插拔选项，以便后期在线升配置

![1006](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1006.png)

此处为网卡选项，多网卡可自先选择，适配器交换机配置不在此详细说明

![1007](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1007.png)

6. 创建任务成功后，后期也可在虚拟机主界面点击编辑进行修改配置

![1008](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1008.png)

7. 首次点击 `打开电源` 即可弹窗进入系统安装界面

> 点击 Tab，打开 kernel 启动选项，增加 `net.ifnames=0 biosdevname=0` 回车即可安装

![1009](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1009.png)

8. 选择系统默认语言，默认 US 即可

![1010](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1010.png)

9. 下一步配置系统时区&时间，可直接点击地图或自行选择

![1011](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1011.png)

![1012](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1012.png)

10. 一般安装主要配置磁盘、网络、软件，因选择的是最小化安装，可忽略

![1013](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1013.png)

11. 磁盘配置可按图例，默认选择整个磁盘，并由系统自行分配

![1014](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1014.png)

![1015](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1015.png)

![1016](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1016.png)

注意：如上述自动分配不满足需求，可自定义选择 `+ -` 进行分配分区

12. 之后配置网络，打开网络配置默认自动获取 IP，如未有显示请自行检查设备，为了后期维护建议选择静态 IP

![1018](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1018.png)

点击上图示的 `Configure` 进行配置

![1019](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1019.png)

![1020](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1020.png)

配置成功后，会自动显示最新的配置清单，请根据实际环境进行配置

![1021](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1021.png)

13. 最后到达安装界面会有滚动条，并且需要配置 `Root` 用户密码，或创建一个新的个人用户账密

![1022](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1022.png)

![1023](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1023.png)

安装完毕后，点击下图示的 `Reboot` 重启服务器即可登录使用

![1024](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/1024.png)

#### 1.2.2 内核升级

```bash
#!/bin/bash

source /etc/init.d/functions

# 检查脚本运行用户是否为Root
if [ $(id -u) != 0 ];then
        echo -e "\033[31mError! You must be root to run this script! \033[0m"
        exit 1
fi

function update_kernel() {
    # 安装EL源
    rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    rpm -Uvh https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
    if [[ $? -ne 0 ]];then
        echo -e "\033[31mError! EL 源安装失败，请检查是否存在问题！\033[0m"
        exit 1
    fi

    # 查看可提供升级的版本
    yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
    VAR_KERNEL_NAME="kernel-lt"
    read -p "请输入上面列出的版本中你想安装的版本（默认 lt 版本） [lt/ml]: " VAR_VERSION_CHOICE
    if [[ ${VAR_VERSION_CHOICE} == "ml" ]];then
        VAR_KERNEL_NAME="kernel-ml"
    fi

    echo "本次选择升级的版本为：${VAR_KERNEL_NAME}"

    # 升级内核
    yum -y --enablerepo=elrepo-kernel install ${VAR_KERNEL_NAME}
    if [[ $? -ne 0 ]];then
        echo "内核升级失败，请根据报错检查是否存在问题！"
        exit 1
    fi

    # 查看目前版本
    echo "系统当前所安装的内核版本如下："
    awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

    # 选择默认内核版本
    grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg
}

function uninstall_kernel() {
    # 显示内核版本
    echo "系统当前所安装的内核版本如下："
    rpm -qa | grep kernel

    # 提示卸载
    echo "你可以手动卸载旧版本：yum -y remove 包名字，然后重启使用：uname -r 查看升级结果"
}

read -p "是否继续安装升级（默认 y） [y/n]: " VAR_CHOICE
case ${VAR_CHOICE} in
    [yY][eE][sS]|[yY])
    update_kernel
    uninstall_kernel
    ;;
    [nN][oO]|[nN])
    echo "安装升级即将终止..."
    exit
    ;;
  *)
    update_kernel
    uninstall_kernel
esac
```

#### 1.2.3 初始化配置

```bash
#!/bin/bash

source /etc/init.d/functions

# 系统版本
System_Version=$(awk -F. '{print $1}' /etc/redhat-release | awk '{print $NF}')

# 编译工具及开发组件
Software="openssh-server ntp ntpdate make cmake bind-utils glibc gcc gcc-c++ zlib zlib-devel openssl openssl-devel pcre pcre-devel curl rsync gd perl perl-core pcre-devel sysstat man mtr openssl-perl subversion nscd dos2unix"

# 常用工具
Tool="bash-completion bash-completion-extras vim screen lrzsz tree psmisc zip unzip bzip2 gdisk telnet net-tools sysstat iftop lsof iotop htop dstat tcpdump setuptool traceroute"

# 检查脚本运行用户是否为Root
if [ $(id -u) != 0 ];then
	echo -e "\033[31mError! You must be root to run this script! \033[0m"
	exit 1
fi

# 判断版本并配置阿里源
function set_config_yum () {
if [ ${System_Version} -eq 7 ];then
    yum -y install wget yum-utils epel-release
    mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak.$(date +%F_%T)
    wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
    wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
    yum clean all && yum makecache
else
    yum -y install wget yum-utils epel-release
    mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak.$(date +%F_%T)
    wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-6.repo
    wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo
    yum clean all && yum makecache
fi
}

# 安装上面定义的编译&常用工具
function set_install_soft () {
    yum -y install ${Software} ${Tool}

    grep -qw "vim" /etc/profile
    [ $? -ne 0 ] && echo "alias vi=vim" >>/etc/profile
    source /etc/profile >/dev/null
}

# 删除无效用户&用户组
function set_config_user () {
    userdel adm
    userdel lp
    userdel shutdown
    userdel operator
    userdel games
    userdel uucp
    groupdel adm
    groupdel lp
    groupdel games
}

# 修改打开文件数限制
function set_config_open_file () {
    /bin/cp /etc/security/limits.conf /etc/security/limits.conf.bak.$(date +%F_%T)
    cat >> /etc/security/limits.conf << EOF
    *   soft    nofile      65535
    *   hard    nofile      65535
    *   soft    nproc       65535
    *   hard    nproc       65535
EOF
    echo "ulimit -SHn 65535" >> /etc/profile
    echo "ulimit -SHn 65535" >> /etc/rc.local

    /bin/cp /etc/security/limits.d/20-nproc.conf  /etc/security/limits.d/20-nproc.conf.bak.$(date +%F_%T)
    echo -e '*      soft    nproc     65535\nroot   soft    nproc     unlimited' > /etc/security/limits.d/20-nproc.conf
}

# 配置时区及时间同步
function set_config_ntp () {
if [ "`cat /etc/crontab | grep ntpdate`" = "" ]; then
	echo "10 * * * * root /usr/sbin/ntpdate cn.pool.ntp.org >> /var/log/ntpdate.log" >> /etc/crontab
fi
rm -rf /etc/localtime
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ntpdate cn.pool.ntp.org && hwclock -w
}

# 配置远程访问限制
function set_config_sshd () {
    /bin/cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F_%T)
    sed -i 's/\#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
    sed -i 's/\#PermitEmptyPasswords no/PermitEmptyPasswords no/' /etc/ssh/sshd_config
    sed -i "s/\#UseDNS yes/UseDNS no/g" /etc/ssh/sshd_config
    sed -i "s/GSSAPIAuthentication yes/GSSAPIAuthentication no/g" /etc/ssh/sshd_config
    #可选 sed -i 's/\#Port 22/Port 65522/' /etc/ssh/sshd_config
}

# 配置Selinux、Iptables(Firewalld)
function set_close_iptables () {
if [ ${System_Version} -eq 7 ];then
	systemctl stop firewalld.service
	systemctl disable firewalld.service
    /bin/cp /etc/selinux/config /etc/selinux/config.bak.$(date +%F_%T)
	sed -i '/SELINUX/s/enforcing/disabled/g' /etc/selinux/config
	setenforce 0
else
	/etc/init.d/iptables stop
	chkconfig iptables off
    /bin/cp /etc/selinux/config /etc/selinux/config.bak.$(date +%F_%T)
	sed -i '/SELINUX/s/enforcing/disabled/g' /etc/selinux/config
	setenforce 0
fi
}

# 设置密码过期天数&最小长度
function set_config_passwd () {
    /bin/cp /etc/login.defs /etc/login.defs.bak.$(date +%F_%T)
    sed -i "/PASS_MIN_DAYS/s/0/80/" /etc/login.defs
    sed -i "/PASS_MIN_LEN/s/5/12/" /etc/login.defs
}

# 内核优化参数
System_Value="
net.ipv4.ip_forward = 1   
net.ipv4.conf.all.rp_filter = 1  
net.ipv4.conf.default.rp_filter = 1 
net.ipv4.conf.all.accept_source_route = 0  
net.ipv4.conf.default.accept_source_route = 0   
kernel.sysrq = 0   
kernel.core_uses_pid = 1  
kernel.msgmnb = 65536 
kernel.msgmax = 65536  
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
###调整系统级别的文件描述符的数量
fs.file-max = 6553500
###调整系统级别的允许线程的数量
kernel.pid_max=1000000
###内存资源使用相关设定
vm.vfs_cache_pressure = 100000
vm.max_map_count = 262144
vm.swappiness = 0
net.core.wmem_default = 8388608   
net.core.rmem_default = 8388608  
net.core.rmem_max = 16777216 
net.core.wmem_max = 16777216 
net.ipv4.tcp_rmem = 4096 8192 4194304 
net.ipv4.tcp_wmem = 4096 8192 4194304    
##应对DDOS攻击,TCP连接建立设置
net.ipv4.tcp_syncookies = 1 
net.ipv4.tcp_synack_retries = 1  
net.ipv4.tcp_syn_retries = 1   
net.ipv4.tcp_max_syn_backlog = 262144 
##应对timewait过高,TCP连接断开设置
net.ipv4.tcp_max_tw_buckets = 6000  
net.ipv4.tcp_tw_recycle = 1  
net.ipv4.tcp_tw_reuse = 1 
net.ipv4.tcp_timestamps = 0   
net.ipv4.tcp_fin_timeout = 30 
net.ipv4.ip_local_port_range = 1024 65000
###TCP keepalived 连接保鲜设置
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_keepalive_intvl = 15 
net.ipv4.tcp_keepalive_probes = 5
###其他TCP相关调节
net.core.somaxconn = 8192 
net.core.netdev_max_backlog = 262144  
net.ipv4.tcp_max_orphans = 3276800    
net.ipv4.tcp_sack = 1  
net.ipv4.tcp_window_scaling = 1
"

# 设置内核优化并执行生效
function set_config_kernel () {
    Sysctl_File="/etc/sysctl.conf"
    /bin/cp /etc/sysctl.conf /etc/sysctl.conf.bak.$(date +%F_%T)

if [ ${System_Version} -eq 6 ];then
	/etc/init.d/sshd restart
	if [ ! -f ${Sysctl_File} ];then
		touch ${Sysctl_File}
	fi
	if [ $(grep -wc "net.ipv4.tcp_max_tw_buckets" ${Sysctl_File}) -eq 0 ];then
		echo "${System_Value}" >${Sysctl_File}		
		/sbin/sysctl -p
	fi
else
	systemctl restart sshd.service
	if [ ! -f ${Sysctl_File} ];then
		touch ${Sysctl_File}
	fi
	if [ $(grep -wc "net.ipv4.tcp_max_tw_buckets" ${Sysctl_File}) -eq 0 ];then
		echo "${System_Value}" >${Sysctl_File}		
		/sbin/sysctl -p
	fi
fi
} 

function main () {
set_config_yum
set_install_soft
set_config_user
set_config_open_file
set_config_ntp
set_config_sshd
set_close_iptables
set_config_passwd
set_config_kernel
}

main
```

## 2. 程序安装

### 2.1 Client 安装

> 如使用 IDEA 或 Vs Code 的可忽略，直接使用 localhost 进行测试；
>
> 所有的安装程序，请自行到相关网站进行下载并上传服务器，版本&安装步骤请参考下述操作；

#### 2.1.1 安装 Java

```bash
# 解压程序至指定目录
$ tar -xf jdk-8u331-linux-x64.tar.gz -C /opt/
$ mv /opt/jdk1.8.0_331 /opt/java

# 配置环境变量
$ vim /etc/profile
# java
JAVA_HOME=/opt/java
JRE_HOME=/opt/java/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH

$ source /etc/profile
$ ln -s /opt/java/bin/java /usr/bin/java
```

#### 2.1.2 安装 Maven

```bash
# 解压程序至指定目录
$ tar -xf apache-maven-3.8.5-bin.tar.gz -C /opt/
$ mv /opt/apache-maven-3.8.5 /opt/maven

# 配置环境变量
$ vim /etc/profile
# maven
export MAVEN_HOME=/opt/maven
export PATH=${JAVA_HOME}/bin:${MAVEN_HOME}/bin:$PATH

$ source /etc/profile
```

修改 `${MAVEN_HOME}/conf/setting.xml` 配置文件，配置参考 [阿里云云效 Maven](https://developer.aliyun.com/mvn/guide)

```xml
$ mkdir -p /opt/maven/repo
$ vim /opt/maven/conf/settings.xml
   // 创建本地仓库目录
   <localRepository>/opt/maven/repo</localRepository>
   
   // 添加阿里云公共仓库
   <mirror>
     <id>aliyunmaven</id>
     <mirrorOf>*</mirrorOf>
     <name>阿里云公共仓库</name>
     <url>https://maven.aliyun.com/repository/public</url>
   </mirror>
```

#### 2.1.3 安装 MySQL

1. 清除旧版本信息

```bash
$ rpm -qa mysql
$ rpm -qa | grep mariadb
$ rpm -e --nodeps mariadb-libs
```

2. 安装相关依赖，解压至指定目录，并安装程序

```bash
# 安装相关依赖
$ yum -y install ncurses-devel cmake libaio-devel openssl-devel gcc gcc-c++ bison

# 编译源码安装
$ tar -xf mysql-boost-5.7.37.tar.gz && cd mysql-5.7.37
$ cmake . \
    -DCMAKE_INSTALL_PREFIX=/opt/mysql \
    -DMYSQL_DATADIR=/opt/mysql/data \
    -DMYSQL_TCP_PORT=3306 \
    -DMYSQL_UNIX_ADDR=/opt/mysql/mysql.sock \
    -DWITH_INNOBASE_STORAGE_ENGINE=1 \
    -DWITH_PARTITION_STORAGE_ENGINE=1 \
    -DWITH_FEDERATED_STORAGE_ENGINE=1 \
    -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
    -DWITH_MYISAM_STORAGE_ENGINE=1 \
    -DENABLED_LOCAL_INFILE=1 \
    -DEXTRA_CHARSETS=all \
    -DDEFAULT_CHARSET=utf8mb4 \
    -DDEFAULT_COLLATION=utf8mb4_general_ci \
    -DWITH_SSL=system \
    -DWITH_BOOST=boost
$ make -j2 && make install
```

3. 创建数据库专用账号，并数据初始化

```bash
# 创建用户并授权目录
$ useradd -r -s /sbin/nologin mysql
$ cd /opt/mysql
$ chown -R mysql:mysql /opt/mysql

# 数据库初始化操作，记录密码
$ bin/mysqld --initialize --user=mysql --basedir=/opt/mysql --datadir=/opt/mysql/data
...
2022-04-03T02:43:31.295939Z 1 [Note] A temporary password is generated for root@localhost: ygnt#l2X(fzx
```

4. 拷贝 mysql.server 脚本到 `/etc/init.d` 目录，编写 MySQL 配置文件，然后启动数据库

```bash
$ cp support-files/mysql.server /etc/init.d/mysql
$ service mysql start
Starting MySQL.Logging to '/usr/local/mysql/data/Dev-Pc.err'.
 SUCCESS!

$ vim my.cnf
[mysqld]
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/usr/local/mysql/mysql.sock

$ service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

5. 重置管理员密码并设置安全配置

```bash
# 重置后记得测试是否成功登录
$ bin/mysqladmin -uroot password 'newpassword' -p
Enter password:
mysqladmin: [Warning] Using a password on the command line interface can be insecure.
Warning: Since password will be sent to server in plain text, use ssl connection to ensure password safety.

# 根据提示进行安全配置，一般第一项回车跳过外，其他默认 Yes
$ bin/mysql_secure_installation
```

6. 添加服务至开机启动并配置环境变量

```bash
$ chkconfig --add mysql
$ chkconfig mysql on

$ vim /etc/profile
# mysql
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin

$ source /etc/profile
```

#### 2.1.4 安装 Redis

1. 检查并升级依赖环境

```bash
# 默认gcc版本过低无法安装reids 6.0以上版本
$ gcc -v 
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)

$ yum -y install centos-release-scl scl-utils-build
$ yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
$ scl enable devtoolset-9 bash

# 注意：scl命令启用只是临时的，退出xshell或者重启就会恢复到原来的gcc版本。如果要长期生效的话，执行如下：
$ echo "source /opt/rh/devtoolset-9/enable" >>/etc/profile
$ source /etc/profile
```

2. 解压至指定目录，并编译安装

```bash
$ tar -xf redis-6.2.6.tar.gz
$ cd redis-6.2.6
$ make && make install PREFIX=/opt/redis
$ mkdir -p /opt/redis/etc /opt/redis/log && cp redis.conf /opt/redis/etc/
```

3. 创建开机启动服务

```bash
$ vim /usr/lib/systemd/system/redis.service
[Unit]
Description=Redis
After=network.target

[Service]
# Type=forking
PIDFile=/var/run/redis_6379.pid
ExecStart=/opt/redis/bin/redis-server /opt/redis/etc/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target

$ systemctl daemon-reload && systemctl enable --now redis && systemctl status redis

# 创建redis命令软链接
$ ln -s /opt/redis/bin/redis-cli /usr/bin/redis
$ echo 'export PATH=/opt/redis/bin:$PATH' >> /etc/profile
$ source /etc/profile
```

#### 2.1.5 安装 NodeJS

```bash
# 解压至指定目录并配置环境变量
$ tar -xf node-v16.14.2-linux-x64.tar.xz -C /opt && cd /opt
$ mv node-v16.14.2-linux-x64 nodejs

$ echo 'export PATH=/opt/nodejs/bin:$PATH' >> /etc/profile
$ source /etc/profile

$ ln -s /opt/nodejs/bin/npm  /usr/local/bin/
$ ln -s /opt/nodejs/bin/node /usr/local/bin/

# 配置淘宝源
$ npm config set registry http://registry.npmmirror.com
$ npm config get registry http://registry.npmmirror.com
```

#### 2.1.6 安装 Nginx

1. 安装依赖包

```bash
$ yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

2. 解压至指定目录，并编译安装

```bash
$ tar -xf nginx-1.21.4.tar.gz && cd nginx-1.21.4
# 测试环境暂不指定安装参数，后续可自行添加编译
$ ./configure --prefix=/opt/nginx 
$ make && make install

# 配置环境变量
$ echo 'export PATH=/opt/nginx/sbin:$PATH' >> /etc/profile
$ source /etc/profile
```

3. 配置开机启动服务

```bash
$ vim /lib/systemd/system/nginx.service
[Unit]
Description=nginx
After=network.target

[Service]
Type=forking
ExecStart=/opt/nginx/sbin/nginx
ExecReload=/opt/nginx/sbin/nginx -s reload
ExecStop=/opt/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target

$ systemctl enable --now nginx && systemctl status nginx
```

#### 2.1.7 安装 Nacos

```bash
# 解压至指定目录
$ tar -xvf nacos-server-2.0.3.tar.gz -C /opt/
```

1. 创建数据库，并初始化 [SQL 数据源](https://github.com/alibaba/nacos/blob/master/distribution/conf/nacos-mysql.sql)

```mysql
mysql> create database nacos_db character set utf8 collate utf8_bin;
mysql> create user 'nacos'@'%' identified by 'nacos';
mysql> grant all privileges on nacos_db.* to 'nacos'@'%';

# 切换数据库，执行nacos初始化数据脚本
mysql> use nacos_db;
mysql> source /opt/nacos/conf/nacos-mysql.sql;
mysql> flush privileges;
```

2. 修改 Nacos 的 `conf/application.properties` 配置文件

```bash
#*************** Config Module Related Configurations ***************#
### If use MySQL as datasource:
spring.datasource.platform=mysql

### Count of DB:
db.num=1

### Connect URL of DB:
# 注意：数据库地址、库名、时区的修改
db.url.0=jdbc:mysql://188.188.4.44:3306/nacos_db?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=Asia/Shanghai
db.user.0=nacos
db.password.0=nacos
```

3. 修改默认的启动方式，改成 `standalone` 单机模式

```bash
$ vim ./nacos/bin/startup.sh
# 将55行的MODE改成standalone,单机启动
 54 export SERVER="nacos-server" 
 55 export MODE="standalone"
```

4. 启动服务测试，浏览器访问 Nacos（默认账密 nacos/nacos）

```bash
$ sh /opt/nacos/bin/startup.sh
$ tail -f /opt/nacos/logs/start.out
```

5. 创建专用用户，并配置开机自启服务

```bash
# 创建用户并授权目录
$ useradd -s /sbin/nologin -M nacos
$ chown -R nacos:nacos /usr/local/nacos

# 创建service文件，定义java路径
$ cat > /usr/lib/systemd/system/nacos.service <<EOF
[Unit]
Description=nacos
After=network.target

[Service]
Type=forking
Environment="JAVA_HOME=/opt/java"
ExecStart=/opt/nacos/bin/startup.sh
ExecReload=/opt/nacos/bin/shutdown.sh
ExecStop=/opt/nacos/bin/shutdown.sh
PrivateTmp=true
User=nacos
Group=nacos

[Install]
WantedBy=multi-user.target
EOF

$ systemctl daemon-reload && systemctl enable --now nacos.service && systemctl status nacos
```

打开浏览器 http://188.188.4.44 默认账密 `nacos/nacos`

#### 2.1.8 安装 Sentinel

1. 官方有提供下载地址，根据需要下载需要的版本，可参考 [官方 Wiki](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)

```bash
# 创建程序目录，并下载启动服务
$ mkdir /usr/local/sentinel/{bin,log}
$ wget https://github.com/alibaba/Sentinel/releases/download/1.8.1/sentinel-dashboard-1.8.1.jar

# 指定参数启动
$ java -Dserver.port=8718 -Dcsp.sentinel.dashboard.server=localhost:8718 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.1.jar
```

2. 配置开机启动服务

```bash
# 开启服务
$ vim /opt/sentinel/bin/startup.sh
#!/bin/bash
nohup java -Dserver.port=8718 -Dcsp.sentinel.dashboard.server=localhost:8718 -Dproject.name=sentinel-dashboard -Dcsp.sentinel.log.dir=/usr/local/sentinel/log -jar /usr/local/sentinel/sentinel-dashboard-1.8.1.jar > /usr/local/sentinel/log/sentinel.log 2>&1 &
echo $! > /var/run/sentinel.pid

# 关闭服务
$ vim /opt/sentinel/bin/shutdown.sh
#!/bin/bash
kill -9 `cat /var/run/sentinel.pid`

$ chmod +x startup.sh shutdown.sh
```

```bash
$ vim /usr/lib/systemd/system/sentinel.service
[Unit]
Description=service for sentinel
After=syslog.target network.target remote-fs.target nss-lookup.target
     
[Service]
Type=forking
Environment="JAVA_HOME=/opt/java"
ExecStart=/opt/sentinel/bin/startup.sh
ExecStop=/opt/sentinel/bin/shutdown.sh
PrivateTmp=true
     
[Install]
WantedBy=multi-user.target

$ systemctl daemon-reload && systemctl enable --now sentinel
$ systemctl status sentinel
$ systemctl list-units --type=service
```

打开浏览器 http://188.188.4.44:8718 默认账密 `sentinel/sentinel`

#### 2.1.9 安装 代码测试

1. 从 git 仓库 clone 下来最新代码 [Ruoyi Gitee 地址](https://gitee.com/y_project/RuoYi-Cloud)

```bash
$ cd /opt && git clone https://gitee.com/y_project/RuoYi-Cloud.git
```

2. 创建数据库并导入数据表数据

```mysql
# 创建两个数据库
mysql> CREATE DATABASE `ry-cloud` CHARACTER SET utf8 COLLATE utf8_general_ci;
mysql> CREATE DATABASE `ry-config` CHARACTER SET utf8 COLLATE utf8_general_ci;

# 导入数据库表
mysql> use ry-cloud;
mysql> source /opt/RuoYi-Cloud/sql/quartz.sql;
mysql> source /opt/RuoYi-Cloud/sql/ry_20210908.sql;
```

使用客户端工具 navicat 连到上面创建的 `nacos_db` 数据库，导入表数据

```mysql
insert into config_info(id, data_id, group_id, content, md5, gmt_create, gmt_modified, src_user, src_ip, app_name, tenant_id, c_desc, c_use, effect, type, c_schema) values 
(1,'application-dev.yml','DEFAULT_GROUP','spring:\n  autoconfigure:\n    exclude: com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure\n  mvc:\n    pathmatch:\n      matching-strategy: ant_path_matcher\n\n# feign 配置\nfeign:\n  sentinel:\n    enabled: true\n  okhttp:\n    enabled: true\n  httpclient:\n    enabled: false\n  client:\n    config:\n      default:\n        connectTimeout: 10000\n        readTimeout: 10000\n  compression:\n    request:\n      enabled: true\n    response:\n      enabled: true\n\n# 暴露监控端点\nmanagement:\n  endpoints:\n    web:\n      exposure:\n        include: \'*\'\n','aaa73b809cfd4d0058893aa13da57806','2020-05-20 12:00:00','2022-04-24 10:26:34','nacos','0:0:0:0:0:0:0:1','','','通用配置','null','null','yaml','null'),
(2,'ruoyi-gateway-dev.yml','DEFAULT_GROUP','spring:\n  redis:\n    host: localhost\n    port: 6379\n    password: \n  cloud:\n    gateway:\n      discovery:\n        locator:\n          lowerCaseServiceId: true\n          enabled: true\n      routes:\n        # 认证中心\n        - id: ruoyi-auth\n          uri: lb://ruoyi-auth\n          predicates:\n            - Path=/auth/**\n          filters:\n            # 验证码处理\n            - CacheRequestFilter\n            - ValidateCodeFilter\n            - StripPrefix=1\n        # 代码生成\n        - id: ruoyi-gen\n          uri: lb://ruoyi-gen\n          predicates:\n            - Path=/code/**\n          filters:\n            - StripPrefix=1\n        # 定时任务\n        - id: ruoyi-job\n          uri: lb://ruoyi-job\n          predicates:\n            - Path=/schedule/**\n          filters:\n            - StripPrefix=1\n        # 系统模块\n        - id: ruoyi-system\n          uri: lb://ruoyi-system\n          predicates:\n            - Path=/system/**\n          filters:\n            - StripPrefix=1\n        # 文件服务\n        - id: ruoyi-file\n          uri: lb://ruoyi-file\n          predicates:\n            - Path=/file/**\n          filters:\n            - StripPrefix=1\n\n# 安全配置\nsecurity:\n  # 验证码\n  captcha:\n    enabled: true\n    type: math\n  # 防止XSS攻击\n  xss:\n    enabled: true\n    excludeUrls:\n      - /system/notice\n  # 不校验白名单\n  ignore:\n    whites:\n      - /auth/logout\n      - /auth/login\n      - /auth/register\n      - /*/v2/api-docs\n      - /csrf\n','2f5a6b5a4ccf20b5801c5cf842456ec6','2020-05-14 14:17:55','2021-07-30 09:07:14',NULL,'0:0:0:0:0:0:0:1','','','网关模块','null','null','yaml','null'),
(3,'ruoyi-auth-dev.yml','DEFAULT_GROUP','spring: \r\n  redis:\r\n    host: localhost\r\n    port: 6379\r\n    password: \r\n','b7354e1eb62c2d846d44a796d9ec6930','2020-11-20 00:00:00','2021-02-28 21:06:58',NULL,'0:0:0:0:0:0:0:1','','','认证中心','null','null','yaml','null'),
(4,'ruoyi-monitor-dev.yml','DEFAULT_GROUP','# spring\r\nspring: \r\n  security:\r\n    user:\r\n      name: ruoyi\r\n      password: 123456\r\n  boot:\r\n    admin:\r\n      ui:\r\n        title: 若依服务状态监控\r\n','d8997d0707a2fd5d9fc4e8409da38129','2020-11-20 00:00:00','2020-12-21 16:28:07',NULL,'0:0:0:0:0:0:0:1','','','监控中心','null','null','yaml','null'),
(5,'ruoyi-system-dev.yml','DEFAULT_GROUP','# spring配置\r\nspring: \r\n  redis:\r\n    host: localhost\r\n    port: 6379\r\n    password: \r\n  datasource:\r\n    druid:\r\n      stat-view-servlet:\r\n        enabled: true\r\n        loginUsername: admin\r\n        loginPassword: 123456\r\n    dynamic:\r\n      druid:\r\n        initial-size: 5\r\n        min-idle: 5\r\n        maxActive: 20\r\n        maxWait: 60000\r\n        timeBetweenEvictionRunsMillis: 60000\r\n        minEvictableIdleTimeMillis: 300000\r\n        validationQuery: SELECT 1 FROM DUAL\r\n        testWhileIdle: true\r\n        testOnBorrow: false\r\n        testOnReturn: false\r\n        poolPreparedStatements: true\r\n        maxPoolPreparedStatementPerConnectionSize: 20\r\n        filters: stat,slf4j\r\n        connectionProperties: druid.stat.mergeSql\\=true;druid.stat.slowSqlMillis\\=5000\r\n      datasource:\r\n          # 主库数据源\r\n          master:\r\n            driver-class-name: com.mysql.cj.jdbc.Driver\r\n            url: jdbc:mysql://localhost:3306/ry-cloud?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8\r\n            username: root\r\n            password: password\r\n          # 从库数据源\r\n          # slave:\r\n            # username: \r\n            # password: \r\n            # url: \r\n            # driver-class-name: \r\n      # seata: true    # 开启seata代理，开启后默认每个数据源都代理，如果某个不需要代理可单独关闭\r\n\r\n# seata配置\r\nseata:\r\n  # 默认关闭，如需启用spring.datasource.dynami.seata需要同时开启\r\n  enabled: false\r\n  # Seata 应用编号，默认为 ${spring.application.name}\r\n  application-id: ${spring.application.name}\r\n  # Seata 事务组编号，用于 TC 集群名\r\n  tx-service-group: ${spring.application.name}-group\r\n  # 关闭自动代理\r\n  enable-auto-data-source-proxy: false\r\n  # 服务配置项\r\n  service:\r\n    # 虚拟组和分组的映射\r\n    vgroup-mapping:\r\n      ruoyi-system-group: default\r\n  config:\r\n    type: nacos\r\n    nacos:\r\n      serverAddr: 127.0.0.1:8848\r\n      group: SEATA_GROUP\r\n      namespace:\r\n  registry:\r\n    type: nacos\r\n    nacos:\r\n      application: seata-server\r\n      server-addr: 127.0.0.1:8848\r\n      namespace:\r\n\r\n# mybatis配置\r\nmybatis:\r\n    # 搜索指定包别名\r\n    typeAliasesPackage: com.ruoyi.system\r\n    # 配置mapper的扫描，找到所有的mapper.xml映射文件\r\n    mapperLocations: classpath:mapper/**/*.xml\r\n\r\n# swagger配置\r\nswagger:\r\n  title: 系统模块接口文档\r\n  license: Powered By ruoyi\r\n  licenseUrl: https://ruoyi.vip','ac8913dee679e65bb7d482df5f267d4e','2020-11-20 00:00:00','2021-01-27 10:42:25',NULL,'0:0:0:0:0:0:0:1','','','系统模块','null','null','yaml','null'),
(6,'ruoyi-gen-dev.yml','DEFAULT_GROUP','# spring配置\r\nspring: \r\n  redis:\r\n    host: localhost\r\n    port: 6379\r\n    password: \r\n  datasource: \r\n    driver-class-name: com.mysql.cj.jdbc.Driver\r\n    url: jdbc:mysql://localhost:3306/ry-cloud?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8\r\n    username: root\r\n    password: password\r\n\r\n# mybatis配置\r\nmybatis:\r\n    # 搜索指定包别名\r\n    typeAliasesPackage: com.ruoyi.gen.domain\r\n    # 配置mapper的扫描，找到所有的mapper.xml映射文件\r\n    mapperLocations: classpath:mapper/**/*.xml\r\n\r\n# swagger配置\r\nswagger:\r\n  title: 代码生成接口文档\r\n  license: Powered By ruoyi\r\n  licenseUrl: https://ruoyi.vip\r\n\r\n# 代码生成\r\ngen: \r\n  # 作者\r\n  author: ruoyi\r\n  # 默认生成包路径 system 需改成自己的模块名称 如 system monitor tool\r\n  packageName: com.ruoyi.system\r\n  # 自动去除表前缀，默认是false\r\n  autoRemovePre: false\r\n  # 表前缀（生成类名不会包含表前缀，多个用逗号分隔）\r\n  tablePrefix: sys_\r\n','8c79f64a4cca9b821a03dc8b27a2d8eb','2020-11-20 00:00:00','2021-01-26 10:36:45',NULL,'0:0:0:0:0:0:0:1','','','代码生成','null','null','yaml','null'),
(7,'ruoyi-job-dev.yml','DEFAULT_GROUP','# spring配置\r\nspring: \r\n  redis:\r\n    host: localhost\r\n    port: 6379\r\n    password: \r\n  datasource:\r\n    driver-class-name: com.mysql.cj.jdbc.Driver\r\n    url: jdbc:mysql://localhost:3306/ry-cloud?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8\r\n    username: root\r\n    password: password\r\n\r\n# mybatis配置\r\nmybatis:\r\n    # 搜索指定包别名\r\n    typeAliasesPackage: com.ruoyi.job.domain\r\n    # 配置mapper的扫描，找到所有的mapper.xml映射文件\r\n    mapperLocations: classpath:mapper/**/*.xml\r\n\r\n# swagger配置\r\nswagger:\r\n  title: 定时任务接口文档\r\n  license: Powered By ruoyi\r\n  licenseUrl: https://ruoyi.vip\r\n','d6dfade9a2c93c463ae857cd503cb172','2020-11-20 00:00:00','2021-01-26 10:36:04',NULL,'0:0:0:0:0:0:0:1','','','定时任务','null','null','yaml','null'),
(8,'ruoyi-file-dev.yml','DEFAULT_GROUP','# 本地文件上传    \r\nfile:\r\n    domain: http://127.0.0.1:9300\r\n    path: D:/ruoyi/uploadPath\r\n    prefix: /statics\r\n\r\n# FastDFS配置\r\nfdfs:\r\n  domain: http://8.129.231.12\r\n  soTimeout: 3000\r\n  connectTimeout: 2000\r\n  trackerList: 8.129.231.12:22122\r\n\r\n# Minio配置\r\nminio:\r\n  url: http://8.129.231.12:9000\r\n  accessKey: minioadmin\r\n  secretKey: minioadmin\r\n  bucketName: test','5382b93f3d8059d6068c0501fdd41195','2020-11-20 00:00:00','2020-12-21 21:01:59',NULL,'0:0:0:0:0:0:0:1','','','文件服务','null','null','yaml','null'),
(9,'sentinel-ruoyi-gateway','DEFAULT_GROUP','[\r\n    {\r\n        \"resource\": \"ruoyi-auth\",\r\n        \"count\": 500,\r\n        \"grade\": 1,\r\n        \"limitApp\": \"default\",\r\n        \"strategy\": 0,\r\n        \"controlBehavior\": 0\r\n    },\r\n	{\r\n        \"resource\": \"ruoyi-system\",\r\n        \"count\": 1000,\r\n        \"grade\": 1,\r\n        \"limitApp\": \"default\",\r\n        \"strategy\": 0,\r\n        \"controlBehavior\": 0\r\n    },\r\n	{\r\n        \"resource\": \"ruoyi-gen\",\r\n        \"count\": 200,\r\n        \"grade\": 1,\r\n        \"limitApp\": \"default\",\r\n        \"strategy\": 0,\r\n        \"controlBehavior\": 0\r\n    },\r\n	{\r\n        \"resource\": \"ruoyi-job\",\r\n        \"count\": 300,\r\n        \"grade\": 1,\r\n        \"limitApp\": \"default\",\r\n        \"strategy\": 0,\r\n        \"controlBehavior\": 0\r\n    }\r\n]','9f3a3069261598f74220bc47958ec252','2020-11-20 00:00:00','2020-11-20 00:00:00',NULL,'0:0:0:0:0:0:0:1','','','限流策略','null','null','json','null');
```
3. 在 Nacos 上修改配置文件，将 MySQL、Redis 的配置改成上述配置链接方式

```yaml
# 选择ruoyi-system-dev.yml，根据自身的配置进行修改成新建的数据库和账密
      datasource:
          # 主库数据源
          master:
            driver-class-name: com.mysql.cj.jdbc.Driver
            url: jdbc:mysql://188.188.4.44:3306/ry-cloud?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
            username: root
            password: newpassword
            
# 选择ruoyi-gen-dev.yml，根据自身的配置进行修改成新建的数据库和账密
spring: 
  redis:
    host: localhost
    port: 6379
    password: 
  datasource: 
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://188.188.4.44:3306/ry-cloud?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
    username: root
    password: newpassword
    
# 选择ruoyi-job-dev.yml，根据自身的配置进行修改成新建的数据库和账密
spring: 
  redis:
    host: localhost
    port: 6379
    password: 
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://188.188.4.44:3306/ry-cloud?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
    username: root
    password: newpassword
```

4. 编译打包并启动测试

```bash
$ mkdir -p /opt/app && cd /opt/RuoYi-Cloud
$ mvn clean install
```

等待很长一段时间后出现 BUILD SUCCESS 表示成功，然后将打包好的 jar 包统一放在一起，挨个启动即可

```bash
# 网关服务
$ cp ruoyi-gateway/target/ruoyi-gateway.jar /opt/app
# 权限服务
$ cp ruoyi-auth/target/ruoyi-auth.jar /opt/app
# 管理后台
$ cp ruoyi-modules/ruoyi-system/target/ruoyi-modules-system.jar /opt/app
# 监控模块
$ cp ruoyi-visual/ruoyi-monitor/target/ruoyi-visual-monitor.jar /opt/app
# 代码生成
$ cp ruoyi-modules/ruoyi-gen/target/ruoyi-modules-gen.jar /opt/app
# 定时任务
$ cp ruoyi-modules/ruoyi-job/target/ruoyi-modules-job.jar /opt/app
# 文件模块
$ cp ruoyi-modules/ruoyi-file/target/ruoyi-modules-file.jar /opt

# 拷贝完成后进入目录挨个启动
$ nohup java -jar ruoyi-gateway.jar &
$ nohup java -jar ruoyi-auth.jar &
$ nohup java -jar ruoyi-modules-system.jar &
# 以上三个是必须启动的，后面四个可选
$ nohup java -jar ruoyi-visual-monitor.jar &
$ nohup java -jar ruoyi-modules-gen.jar &
$ nohup java -jar ruoyi-modules-job.jar &
$ nohup java -jar ruoyi-modules-file.jar &
```

5. 配置前端服务，并打包编译

```bash
$ cd RuoYi-Cloud/ruoyi-ui
# 如需更改端口，可修改以下js配置文件
$ vim vue.config.js
const name = process.env.VUE_APP_TITLE || '若依管理系统' // 网页标题
const port = process.env.port || process.env.npm_config_port || 80 // 端口
```

```bash
# 清除之前编译记录并安装npm环境、清除缓存、升级
$ sudo rm -rf node_modules package-lock.json && npm install
$ npm cache clean --force
$ npm install -g npm@8.8.0

# build:prod为指定编译哪个配置文件，可自行修改
$ npm run build:prod
```

6. 将打包生成后的 `dist/*` 文件发布到 Nginx 目录，配置 Nginx 并重新加载服务

```bash
$ vim /opt/nginx/conf/nginx.conf
server {
        listen       80;
        server_name  localhost;

        location / {
            root   html/ruoyi-cloud;
            try_files $uri $uri/ /index.html;
            index  index.html index.htm;
        }

        location /prod-api/ {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header REMOTE-HOST $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://localhost:8080/;
     }

        location /dev-api/ {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header REMOTE-HOST $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://localhost:8080/;
     }
}

# 保存后生新加载一下
$ nginx -s reload
```

7. 访问页面测试，如正常访问前端并可登录即代表部署成功，打开浏览器 http://188.188.4.44 默认账密 `ruoyi/123456`

> 注：为了后期的测试，请将刚执行的 Jar 程序进行关闭运行，并上传项目代码至后面的 Gitlab 仓库

### 2.2 Harbor 镜像仓库

#### 2.2.1 安装 Docker

1. 安装依赖包，配置阿里源

```bash
$ yum -y install yum-utils device-mapper-persistent-data lvm2
$ yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
$ yum makecache fast
```

2. 安装最新版本或自行选择版本，配置安全设置并启动

```bash
$ yum list docker-ce --showduplicates | sort -r
$ yum install docker-ce docker-ce-cli containerd.io -y

$ vim /lib/systemd/system/docker.service
[Service]
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT

# 启动并加入开机启动
$ systemctl start docker && systemctl enable docker
```

3. 配置加速器

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
      "https://registry.docker-cn.com"
      ],
  "log-driver": "json-file",
  "log-opts" : 
    {
      "max-size": "500m","max-file":"3"
    }
}
EOF

# 重启 docker 服务
$ systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```

#### 2.2.2 安装 Docker-compose

```bash
# 建议离线安装，自行到Github进行下载上传
$ wget https://github.com/docker/compose/releases/download/v2.5.1/docker-compose-linux-x86_64
$ mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
$ docker-compose --version
```

#### 2.2.3 安装 Harbor

1. 下载上传，解压至指定目录并创建证书目录

```bash
$ tar -xf harbor-offline-installer-v2.4.1.tgz -C /opt
$ mkdir -p /opt/harbor/cert && cd /opt/harbor/cert
```

2. 生成证书颁发机构的证书&服务器证书

```bash
# 生成CA证书私钥
$ openssl genrsa -out ca.key 4096

# 生成CA证书
$ openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=DongGuan/L=DongGuan/O=example/OU=Personal/CN=harbor.yuikuen.top" \
 -key ca.key \
 -out ca.crt
 
 # 生成私钥
$ openssl genrsa -out harbor.yuikuen.top.key 4096
# 生成证书签名请求(CSR)
$ openssl req -sha512 -new \
    -subj "/C=CN/ST=DongGuan/L=DongGuan/O=example/OU=Personal/CN=harbor.yuikuen.top" \
    -key harbor.yuikuen.top.key \
    -out harbor.yuikuen.top.csr

# 生成x509 v3扩展文件
$ cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.yuikuen.top
DNS.2=harbor.yuikuen
DNS.3=harbor.yuikuen.top
EOF
//无论是使用 FQDN 还是 IP 地址连接到 Harbor 主机，都必须创建此文件，以便为 Harbor 主机生成符合主题备用名称 (SAN) 和 x509 v3 的证书扩展要求

# 使用v3.ext文件为Harbor主机生成证书，将CRS和CRT文件名替换为Harbor主机名
$ openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.yuikuen.top.csr \
    -out harbor.yuikuen.top.crt
```

3. 提供证书（Docker 需要将 CRT 文件作为 CA 证书，CERT 文件作为客户端证书）

```bash
$ openssl x509 -inform PEM -in harbor.yuikuen.top.crt -out harbor.yuikuen.top.cert

$ mkdir -p /etc/docker/certs.d/harbor.yuikuen.top
$ cp ca.crt *.top.cert *.top.key /etc/docker/certs.d/harbor.yuikuen.top/

$ systemctl daemon-reload && systemctl restart docker
```

4. 修改配置文件，并执行脚本测试（若是采用 http 方式搭建，可忽视 https 的配置部分，将其注释掉）

```bash
$ cp harbor.yml.tmpl harbor.yml && vim harbor.yml
$ vim harbor.yml
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

执行预备脚本`./prepare` ，待测试完出现 `Successfully called func: create_root_cert`表示可正常部署

```bash
$ ./install.sh --with-chartmuseum --with-notary --with-trivy
`--with-chartmuseum` 是安装chart仓库，不使用helm可不添加该参数
`--with-notary` 含义是启用镜像签名，必须是 https 才可以，否则会报错`ERROR:root:Error: the protocol must be https when Harbor is deployed with Notary`
```

5. 设置开机启动服务

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
ExecStart=/usr/local/bin/docker-sssscompose -f  /opt/harbor/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /opt/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target
```

打开浏览器 https://harbor.yuikuen.top 默认账密 `admin/Harbor12345`

### 2.3 SonarQube 代码审查

> 部署不同版本，需要的环境支持也不一样，请自行参考官方的 [环境要求](https://docs.sonarqube.org/8.9/requirements/requirements/)

#### 2.3.1 安装 Java

```bash
$ yum install -y java-11-openjdk java-11-openjdk-devel

# 查看并确认具体安装路径
$ which java
$ ls -lr /usr/bin/java
$ ls -lrt /etc/alternatives/java
/etc/alternatives/java -> /usr/lib/jvm/java-11-openjdk-11.0.13.0.8-1.el7_9.x86_64/bin/java

# 配置环境变量
$ vim /etc/profile
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.13.0.8-1.el7_9.x86_64
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
$ source /etc/profile
```

#### 2.3.2 安装 PostgreSQL

1. 下载 RPM 文件安装并初始化

```bash
$ yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm -y
$ yum install postgresql10-contrib postgresql10-server -y
$ /usr/pgsql-10/bin/postgresql-10-setup initdb
```

2. 修改配置并开启访问权限

```bash
$ cp /var/lib/pgsql/10/data/pg_hba.conf{,.bak}

# 将 peer、ident 改为 trust ，改了6行
$ vim /var/lib/pgsql/10/data/pg_hba.conf
# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust

$ systemctl enable --now postgresql-10.service
```

3. 创建用户及数据库

```sql
$ su - postgres
Last login: Mon Nov  1 11:14:43 CST 2021 on pts/0
-bash-4.2$ psql
psql (10.18)
Type "help" for help.
postgres=# CREATE DATABASE sonar TEMPLATE template0 ENCODING 'utf8' ;
postgres=# create user sonar;
postgres=# alter user sonar with password 'sonar';
postgres=# alter role sonar createdb;
postgres=# alter role sonar superuser;
postgres=# alter role sonar createrole;
postgres=# alter database sonar owner to sonar;
postgres=# \q
-bash-4.2$ exit
```

#### 2.3.3 安装 SonarQube

1. 添加用户，解压至指定目录并授权

```bash
$ adduser sonar
$ wget -c https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.9.6.50800.zip
$ unzip sonarqube-8.9.6.50800.zip
$ mv sonarqube-8.9.6.50800 /opt/sonarqube
$ chown -R sonar:sonar /opt/sonarqube/
```

2. 修改配置，链接 Postgresql

```bash
$ vim /opt/sonarqube/conf/sonar.properties
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:postgresql://localhost/sonar
sonar.jdbc.maxActive=60
sonar.jdbc.maxIdle=5
sonar.jdbc.minIdle=2
sonar.jdbc.maxWait=5000
sonar.jdbc.minEvictableIdleTimeMillis=600000
sonar.jdbc.timeBetweenEvictionRunsMillis=30000
sonar.jdbc.removeAbandoned=true
sonar.jdbc.removeAbandonedTimeout=60
```

3. 配置环境变量

```bash
$ vim /etc/profile
# sonarqube
export SONAR_HOME=/opt/sonarqube
export SONAR_RUNNER_HOME=/opt/sonar-scanner
export PATH=$PATH:$SONAR_RUNNER_HOME/bin
export PATH=$PATH:$SONAR_HOME/bin
$ source /etc/profile
```

4. 启动服务并配置开机启动服务

```bash
$ vim /etc/systemd/system/sonar.service
[Unit]
Description=SonarQube Server
After=syslog.target network.target
 
[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop= /opt/sonarqube/bin/linux-x86-64/sonar.sh stop
LimitNOFILE=65536
LimitNPROC=4096
User=sonar
Group=sonar
Restart=on-failure
 
[Install]
WantedBy=multi-user.target

$ systemctl restart sonar.service
$ systemctl enable --now sonar.service && systemctl status sonar.service
```

#### 2.3.4 程序汉化

> 汉化请自行到 Github 下载相应 [release](https://github.com/xuhuisheng/sonar-l10n-zh)，再上传到 `./sonarqube/extensions/plugins`，重启服务即可

SonarQube 汉化包兼容列表如下：

| **SonarQube** | **9.0** | **9.1** | **9.2** | **9.3** |         |         |         |         |         |         |
| ------------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- | ------- |
| sonar-l10n-zh | 9.0     | 9.1     | 9.2     | 9.3     |         |         |         |         |         |         |
| **SonarQube** | **8.0** | **8.1** | **8.2** | **8.3** | **8.4** | **8.5** | **8.6** | **8.7** | **8.8** | **8.9** |
| sonar-l10n-zh | 8.0     | 8.1     | 8.2     | 8.3     | 8.4     | 8.5     | 8.6     | 8.7     | 8.8     | 8.9     |
| **SonarQube** | **7.0** | **7.1** | **7.2** | **7.3** | **7.4** | **7.5** | **7.6** | **7.7** | **7.8** | **7.9** |
| sonar-l10n-zh | 1.20    | 1.21    | 1.22    | 1.23    | 1.24    | 1.25    | 1.26    | 1.27    | 1.28    | 1.29    |
| **SonarQube** | **6.0** | **6.1** | **6.2** | **6.3** | **6.4** | **6.5** | **6.6** | **6.7** |         |         |
| sonar-l10n-zh | 1.12    | 1.13    | 1.14    | 1.15    | 1.16    | 1.17    | 1.18    | 1.19    |         |         |
| **SonarQube** |         |         |         |         | **5.4** | **5.5** | **5.6** |         |         |         |
| sonar-l10n-zh |         |         |         |         | 1.9     | 1.10    | 1.11    |         |         |         |
| **SonarQube** | **4.0** | **4.1** |         |         |         |         |         |         |         |         |
| sonar-l10n-zh | 1.7     | 1.8     |         |         |         |         |         |         |         |         |
| **SonarQube** |         | **3.1** | **3.2** | **3.3** | **3.4** | **3.5** | **3.6** | **3.7** |         |         |
| sonar-l10n-zh |         | 1.0     | 1.1     | 1.2     | 1.3     | 1.4     | 1.5     | 1.6     |         |         |

打开浏览器 http://188.188.4.220:9000 默认账密 `admin/admin`

### 2.4 Gitlab 代码仓库

> 企业内部使用，建议关闭【账户和限制】的用户注册功能

1. 下载依赖环境

```bash
$ yum -y install policycoreutils openssh-server openssh-clients postfix
$ systemctl enable --now sshd postfix
```

2. 下载 Rpm 文件，再直接离线安装

```bash
$ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-14.1.0-ce.0.el7.x86_64.rpm
$ rpm -ivh gitlab-ce-14.1.0-ce.0.el7.x86_64.rpm --force --nodeps
```

3. 修改 `/etc/gitlab/gitlab.rb` 配置文件，并启动服务

```bash
## GitLab URL 可为域名或本机IP
external_url 'http://gitlab.yuikuen.top'

// 邮箱配置 (非必要，自先选择)
### GitLab email server settings
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"           //邮件服务
gitlab_rails['smtp_port'] = 465                        //465默认开启smtp_tls
gitlab_rails['smtp_user_name'] = "sinath@vip.qq.com"   //发件人地址
gitlab_rails['smtp_password'] = "****************"     //邮箱授权码(自行百度)
gitlab_rails['smtp_domain'] = "qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['smtp_pool'] = false

### Email Settings
gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = 'sinath@vip.qq.com' //发件人邮箱
gitlab_rails['gitlab_email_display_name'] = 'sinath'    //发件人名称
user['git_user_email'] = "sinath@vip.qq.com"            //同上

$ gitlab-ctl reconfigure
$ gitlab-ctl restart
```

4. 正常输出日志代表部署成功，打开浏览进行登录，初始管理员用户为 **root**

```bash
# 密码随机生成，可通过下述路径进行查询，注意有效期 24 小时，首次登录需修改 root 密码
$ cat /etc/gitlab/initial_root_password
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: d5WxYmXvXbEOfpzm/LZ6Bibmc+HzNTGpjNQU/3hdbh0=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

打开浏览器 http://188.188.4.221 默认账密 `root/d5WxYmXvXbEOfpzm/LZ6Bibmc+HzNTGpjNQU/3hdbh0=`

### 2.5 Jenkins 发布平台

#### 2.5.1 安装 Java

```bash
# 解压程序至指定目录
$ tar -xf jdk-8u331-linux-x64.tar.gz -C /opt/
$ mv /opt/jdk1.8.0_331 /opt/java

# 配置环境变量
$ vim /etc/profile
# java
JAVA_HOME=/opt/java
JRE_HOME=/opt/java/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH

$ source /etc/profile
$ ln -s /opt/java/bin/java /usr/bin/java
```

#### 2.5.2 安装 Maven

```bash
# 解压程序至指定目录
$ tar -xf apache-maven-3.8.5-bin.tar.gz -C /opt/
$ mv /opt/apache-maven-3.8.5 /opt/maven

# 配置环境变量
$ vim /etc/profile
# maven
export MAVEN_HOME=/opt/maven
export PATH=${JAVA_HOME}/bin:${ MAVEN_HOME}/bin:$PATH

$ source /etc/profile
```

修改 `${MAVEN_HOME}/conf/setting.xml` 配置文件，配置参考 [阿里云云效 Maven](https://developer.aliyun.com/mvn/guide)

```xml
$ mkdir -p /opt/maven/repo
$ vim /opt/maven/conf/settings.xml
   // 创建本地仓库目录
   <localRepository>/opt/maven/repo</localRepository>
   
   // 添加阿里云公共仓库
   <mirror>
     <id>aliyunmaven</id>
     <mirrorOf>*</mirrorOf>
     <name>阿里云公共仓库</name>
     <url>https://maven.aliyun.com/repository/public</url>
   </mirror>
```

#### 2.5.3 安装 Tomcat

```bash
# 解压至指定目录并更名
$ tar -xf apache-tomcat-8.5.73.tar.gz -C /opt/ && cd /opt
$ mv apache-tomcat-8.5.73 tomcat

# 启动服务并检测服务是否正常
$ sh /opt/tomcat/bin/startup.sh 
Using CATALINA_BASE:   /opt/tomcat
Using CATALINA_HOME:   /opt/tomcat
Using CATALINA_TMPDIR: /opt/tomcat/temp
Using JRE_HOME:        /opt/java/jre
Using CLASSPATH:       /opt/tomcat/bin/bootstrap.jar:/opt/tomcat/bin/tomcat-juli.jar
Using CATALINA_OPTS:   
Tomcat started.

$ lsof -i:8080
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    1564 root   55u  IPv4  22515      0t0  TCP *:webcache (LISTEN)
```

配置开机启动服务

```bash
# 为tomcat配置pid
$ vim /opt/tomcat/bin/catalina.sh
# Copy CATALINA_BASE from CATALINA_HOME if not already set
[ -z "$CATALINA_BASE" ] && CATALINA_BASE="$CATALINA_HOME"
CATALINA_PID="$CATALINA_BASE/tomcat.pid"  # 设置pid。一定要加在CATALINA_BASE定义后面，要不然pid会生成到/下面

$ vim /lib/systemd/system/tomcat.service 
[Unit]
Description=Tomcat
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking

Environment="JAVA_HOME=/opt/java"
PIDFile=/opt/tomcat/tomcat.pid
ExecStart=/opt/tomcat/bin/startup.sh
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target

$ kill -9 1564
$ systemctl enable --now tomcat
```

#### 2.5.4 安装 Jenkins

> 下载指定版本的程序到 Tomcat 网站目录，再解压重启服务，根据提示获取密码，安装插件可选择先跳过

```bash
$ cd /opt/tomcat/webapps/ROOT
$ wget https://get.jenkins.io/war-stable/2.332.3/jenkins.war
$ jar –xvf jenkins.war
$ systemctl restart tomcat
```

打开浏览器 http://188.188.4.222:8080 默认账密首次登录自定义

## 3. 发版流程

> 配置的内容主要演示快速部署 SpringCloud 项目，并未具体介绍参数配置

### 3.1 配置 Gitlab

#### 3.1.1 创建用户

> 创建用户的时候，可选择 Regular 或 Admin 类型，后续演示都以此开发人员为例

![image-20220519171749045](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220519171749045.png)

#### 3.1.2 创建项目

> 注意用户的角色和访问权限即可，后续也可在群组【设置】添加成员或修改角色权限

- Guest：可以创建issue、发表评论，不能读写版本库
- Reporter：可以克隆代码，不能提交，QA、PM 可以赋予这个权限
- Developer：可以克隆代码、开发、提交、push，普通开发可以赋予这个权限
- Maintainer：可以创建项目、添加tag、保护分支、添加项目成员、编辑项目，核心开发可以赋予这个 权限
- Owner：可以设置项目访问权限 - Visibility Level、删除项目、迁移项目、管理组成员，开发组组 长可以赋予这个权限

1. 为了区分项目类目，创建群组来管理

![image-20220519172319958](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220519172319958.png)

2. 将之前 Client 演示的源码上传至 Gitlab 仓库。也可直接从 URL 导入仓库，再根据之前的演示进行修改

![image-20220519173444696](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220519173444696.png)

#### 3.1.3 配置凭证

> 为了方便后期代码管理维护，并且安全有效，建议使用 SSH 密钥方式

1. 个人 Win 电脑安装 [Git 程序](https://git-scm.com/)，安装过程在此略过（Linux 配置过程同理）
2. 生成密钥，桌面右键 > `Git Bash Here`  > 输入命令生成

```bash
# 一直回车下一步即可
$ ssh-keygen -t rsa -C "[your_mail]" -f ~/.ssh/[custom_name]
// your_mail 表示你的邮箱地址。
// custom_name 表示公钥私钥的名称。[-f ~/.ssh/xxx]可选，默认名称为id_rsa

$ ls -l ~/.ssh/
total 13
-rw-r--r-- 1 ztnb9 197609 2610 Mar 31 14:35 id_rsa
-rw-r--r-- 1 ztnb9 197609  578 Mar 31 14:35 id_rsa.pub
```

3. 上传密钥至 Gitlab，并测试连接

![image-20220519175200748](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220519175200748.png)

```bash
$ ssh -T git@188.188.4.221
Welcome to GitLab, @yuen!
```

### 3.2 配置 Jenkins

> 各类型项目的发布都依赖着插件及相关应用配置，具体参数介绍及应用方法可查看 Jenkins 相关教程

#### 3.2.1 插件管理

1. 进入安装目录 `/root/.jenkins/updates` 修改配置，否则后期会提示无法连接问题

```bash
$ cd /root/.jenkins/updates
$ sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

2. 更改插件下载地址：Jenkins > Manage Jenkins > Manage Plugins，点击标签 Advanced

> https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

在浏览器输入： http://188.188.4.222:8080/restart ，重启 Jenkins

#### 3.2.2 权限管理

> 用户权限可利用 [Role-based Authorization Strategy](https://plugins.jenkins.io/role-strategy) 插件来管理用户权限

1. 安装插件后，开启全局安全配置：Dashboard > 全局安全配置 > 授权策略切换为"Role-Based Strategy"

![image-20220519181331544](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220519181331544.png)

2. 创建用户 & 角色，并且进行权限划分：(根据实际环境进行配置，此仅供参考)

- 全局角色(baseRole)，只绑定 Overall(全部) 下的 Read 权限，便于新增用户的基本访问权限管理
- 开发人员 Dev、测试人员 Uat、正式发布 Prod

注：如不绑定相应角色的用户为提示 <font color=red>用户名 is missing the Overall/Read permission</font>

![image-20220520171201819](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520171201819.png)

上图实例按用户角色来创建的，仅供参考，可根据实际需求进行定制

3. 点击 `Manage and Assign Roles`，根据用户权限进行分配角色

- Global roles(全局角色)：管理员等高级用户可以创建基于全局的角色
- Item roles(项目角色)：针对某个或者某些项目的角色
- Node roles(节点角色)：节点相关的权限，多为主从时使用

![image-20220520171459114](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520171459114.png)

`No type prefix:yuen` 此报错不影响后续使用，在旧版本显示正常，可能是新版本某插件兼容问题

#### 3.2.3 凭证管理

> 凭证可存储各类账密信息，以便 Jenkins 与其进行交互

1. 安装 `Credentials Binding` 插件
2. 安装后在管理页面就可看到 `Manage Credentials`，并且可以创建各类凭证，凭证类型有 5 种：
   - Username with password：用户名和密码
   - SSH Username with private key：使用 SSH 用户和密钥
   - Secret file：需要保密的文本文件，使用时 Jenkins 会将文件复制到一个临时目录中，再将文件路径设置到一个变量中，等构建结束后，所复制的 Secret file 就会被删除
   - Secret test：需要保存的一个加密的文本串，如钉钉机器人或 Github 的 API Token
   - Certificate：通过上传证书文件的方式
3. 选择"Username with password"，输入 Gitlab 的用户名和密码，点击"确定"  (添加之前搭建 GitLab 所创建的用户)

![image-20220520175125616](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520175125616.png)

4. 测试凭证(http类型)，添加测试项目(为了让 Jenkins 支持从 GitLab 拉取源码，需要安装 Git 插件以及在 CentOS 7上安装 Git 工具)

![image-20220520175355080](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520175355080.png)

直接采用的是 `http 凭证类型` 拉取项目代码构建，简单演示代码拉取构建过程，查看 `./jenkins/workspace` 目录，可查看已从 Gitlab 成功拉取的代码

5. 为安全交互，建议使用 `SSH 密钥` 方式，在 Jenkins 服务器使用 root 用户生成公钥 & 私钥

![image-20220520180837142](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520180837142.png)

```bash
# ssh-keygen -t rsa -C "[your_mail]" -f ~/.ssh/[custom_name]
$ ssh-keygen -t rsa -C "jenkins@example.com"                //一路回车即可
$ ls ~/.ssh/
ls ~/.ssh/
id_rsa  id_rsa.pub
```

6. 把生成的公钥放在 Gitlab 的 SSH Keys 中，并且在 Jenkins 中添加凭证私钥

![image-20220520180304272](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520180304272.png)

**Private Key** 填写 Jenkins 的私钥

![image-20220520180533634](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520180533634.png)

测试项目改成 `SSH` 凭证方式进行拉取，同样构建并查看目录是否成功拉取

![image-20220520181055790](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520181055790.png)

#### 3.2.4 环境配置

> 后期项目需要 Maven 编译打包等操作，需要提前配置环境变量，其他类型项目雷同；

1. Manage Jenkins > Global Tool Configuration > JDK / Maven

![image-20220520181446217](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520181446217.png)

2. Manage Jenkins > Configure System > Global Properties，添加三个全局变量 JAVA_HOME、M2_HOME、PATH+EXTRA

![image-20220520181524425](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520181524425.png)

3. 安装 `Maven` 相关的插件，如 [Maven Integration plugin](https://plugins.jenkins.io/maven-plugin)，再次创建 Job 为 Maven 的项目进行发布

![image-20220520182307537](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220520182307537.png)

#### 3.2.5 项目构建

> Jenkins 可定制构建各类型项目，根据项目内容安装对应插件即可实现构建，本实例重点以声明式流水线 `pipeline` 来讲解

1. 创建 Pipeline 有两种方法：
   - 直接在 Jenkins 的 Web UI 界面中输入脚本；
   - 通过创建 Jenkinsfile 脚本文件至项目源码库根目录下作调用；

**通过 Gitlab 可有效管理 Jenkinsfile 发版的变动，为了更好的管理，推荐从源代码控制(SCM)中直接载入 Jenkinsfile Pipeline**

- 在 Gitlab 项目根目录建立 Jenkinsfile 文件

![image-20220521142822831](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220521142822831.png)

- 再到 Jenkins 项目中引用 **Pipeline Script from SCM**，构建项目时自动读取 Jenkinsfile 文件进行流水线

![image-20220521142653483](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220521142653483.png)

2. 流水线构建可分为：脚本式和声明式，下面以实例讲解

> 两种流水线的执行结果都是一样，可根据任务逻辑的复杂程度进行选择，下述都以官方推荐的声明式来讲解。

- 脚本式流水线官例：

![image-20220521143636817](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220521143636817.png)

- 声明式流水线官例：

![image-20220521144532904](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220521144532904.png)

3. 操作实例：以 `RuoYi Cloud` 为例创建声明式为例，通过官方实例 `hello world` 进行修改代码

**操作方法：Job 创建流水线项目 > 流水线选择声明式 `Pipeline script from SCM` > SCM 选择指定 Git 仓库地址 > Gitlab 项目根目录创建 Jenkinsfile 文件 > 通过 Vs Code Clone 仓库进行修改并实现构建；**

- 创建流水线 Job 任务，选择 `SCM` + `SSH` 进行拉取代码

![image-20220521145433593](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220521145433593.png)

- Gitlab 项目根路径创建 `Jenkinsfile` 文件，以官方实例进行修改

```groovy
// 一般发版流程模式，仅供参考
pipeline {
    agent any

    stages {
        stage('拉取代码') {
            steps {
                echo '拉取代码'
            }
        }
        stage('编译构建') {
            steps {
                echo '编译构建'
            }
        } 
        stage('部署发布') {
            steps {
                echo '部署发布'
            }
        }
    }
}
```

**通过 `流水线语法` 完善代码块，首先进行代码拉取，选择示例步骤`checkout:Check out from version control` 生成脚本**

![image-20220521154408415](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220521154408415.png)

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
    }
}
```

- 测试没问题后，下一步就是编译构建，直接执行 sh 命令进行 maven 打包，之后可在 `workspace` 查看到已成功构建的 Jar 文件

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
        
        // 编译打包
        stage('Code Package') {
            steps {
                // 编译并打包服务工程
                sh "mvn clean package -Dmaven.test.skip=true"
            }
        }
    }
}
```

#### 3.2.6 代码审查

**Jenkins 代码审查过程如下图所示：**

![image-20220524094808501](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524094808501.png)

1. 已安装 SonarQube 服务端，现 Jenkins 调用需配置对应插件方可交互。安装 `SonarQube Scanner` 插件，添加 SonarQube 凭证

> 插件已安装后，首先到 SonarQube > 右上角用户管理 > 安全 > 填写令牌名称(任意值)，创建 Token

**【sonarqube 支持生成用户 token，以便在命令行或者脚本中使用 token 代表账号操作 sonarbue，避免造成账号密码的泄露】**

![image-20220524095251711](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524095251711.png)

![image-20220524095319641](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524095319641.png)

2. Jenkins 配置：Manage Jenkins > Configure System > SonarQube Servers

> 添加刚安装好的 SonarQube 服务器地址及凭证认证信息

![image-20220524100124462](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524100124462.png)

3. 代码扫描功能还需整合 SonarQube Scanner，方可与 SonarQube 进行交互，Manage Jenkins > Global Tool Configuration

![image-20220524100428167](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524100428167.png)

4. 关闭审查结果上传至 SCM 功能

![image-20220524100529296](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524100529296.png)

5. 参考 [SonarQube官方教程](https://docs.sonarqube.org/8.9/analysis/scan/sonarscanner/) 进行设置分析属性

![image-20220524100758908](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524100758908.png)

```
# 官网参考文档
# must be unique in a given SonarQube instance
sonar.projectKey=my:project                //唯一标记，供代码到sonarqube所定义的唯一标记

# --- optional properties ---

# defaults to project key
#sonar.projectName=My project              //项目的名称，一般与key一样
# defaults to 'not provided'
#sonar.projectVersion=1.0                  //版本号
sonar.exclusions=**/test/**,**/target/**   //排除的元素，过滤不去扫描的文件目录
 
# Path is relative to the sonar-project.properties file. Defaults to .
#sonar.sources=.                           //指定sonarqube扫描的路径，一般为根目录，或指定目录/src/man/**

sonar.java.source=1.8                      //jdk的版本
sonar.java.target=1.8

# Encoding of the source code. Default is default system encoding
#sonar.sourceEncoding=UTF-8                //源码的编码格式
```

**`RuoYi-Cloud` 项目共有 6 个，在此以 `ruoyi-gateway` 为例，其它子项目操作同理**

- 在子项目根目录创建文件 `sonar-project.properties`

![image-20220524112731977](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524112731977.png)

```
# must be unique in a given SonarQube instance
sonar.projectKey=ruoyi-gateway

# --- optional properties ---

# defaults to project key
sonar.projectName=ruoyi-gateway
# defaults to 'not provided'
sonar.projectVersion=v0.0.1
sonar.exclusions=**/target/**
sonar.java.binaries=target

# Path is relative to the sonar-project.properties file. Defaults to .
sonar.sources=.

sonar.java.source=1.8
sonar.java.target=1.8

# Encoding of the source code. Default is default system encoding
sonar.sourceEncoding=UTF-8
```

- `RuoYi-Cloud` 子项目共有 6个，需添加参数化-字符、选项参数来区分项目分支和指定子项目

![image-20220524120403084](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524120403084.png)

![image-20220524120443092](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524120443092.png)

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                // 修改分支获取参数"*/${branch}"
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
        
        // 编译打包
        stage('Code Package') {
            steps {
                // 编译公共子工程
                //sh "mvn -f ruoyi-common clean install"
                // 编译并打包指定子项目
                sh "mvn clean package -Dmaven.test.skip=true"
            }
        }
        
        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    // 进入指定子项目
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
   }
}
```

构建完成后，可在 SonarQube 服务端查看代码审查的记录及相关信息

![image-20220524140917742](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524140917742.png)

#### 3.2.7 远端部署

> 下一步就是将 Jar 包部署至对应的服务器上运行，发送远程命令执行脚本运行

1. 首先安装 `Publish over SSH` 插件，配置信任的远端服务器，方法任意选一

密码方式：按信息填写远端服务器的名称、IP、ssh 用户、远程目录和密码即可；

![image-20220521161909157](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220521161909157.png)

密钥方式：将 Jenkins 私钥填在【Jenkins SSH Key】，再将公钥复制到远程服务器的【authorized_keys】，其它填名称、IP、用户和远程目录即可；(若不成功请自行检查各服务器之间的交互性)

![image-20220521161954681](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220521161954681.png)

2. 通过流水线语法，选择 `sshPublisther:Send build artifacts over SSH` 获取语法（内容先不用填，选服务器即可）

![image-20220524143127678](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220524143127678.png)

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
       
        // 编译打包
        stage('Code Package') {
            steps {
                // 编译公共子工程
                //sh "mvn -f ruoyi-common clean install"
                // 编译并打包指定服务工程
                sh "mvn -f ${project_name} clean package -Dmaven.test.skip=true"
            }
        }
        
        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
        
        // 上线发布
        stage('Deploy Server') {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'debug', 
                // 清除远程目录内文件
                transfers: [sshTransfer(cleanRemote: true, 
                excludes: '', execCommand: 'sh /home/yuen/deploy.sh $project_name', execTimeout: 120000, flatten: false, 
                makeEmptyDirs: false, 
                noDefaultExcludes: false, 
                patternSeparator: '[, ]+', 
                remoteDirectory: 'opt/app', remoteDirectorySDF: false, 
                removePrefix: '${project_name}/target', 
                sourceFiles: '${project_name}/target/*.jar')], 
                usePromotionTimestamp: false, 
                useWorkspaceInPromotion: false, 
                verbose: false)])
            }
        }
    }
}
```
**远端服务器脚本**

```bash
#!/bin/bash

# 接收外部参数
project_name=$1
app_path=/opt/app

echo "$project_name"

# 检测现有pid并执行杀死操作
pid=`ps -ef|grep $project_name | grep "java" | awk '{print $2}'`
if [ "$pid" != "" ];then
  echo "already start pid: $pid"
  echo "kill first then start"
  kill -9 $pid
fi

# 启动新的java项目
nohup java -jar $app_path/$project_name.jar >/dev/null 2>&1 &

# 检测新的java项目是否正常启动，记录日志信息
pid=$(ps -ef | grep "java" | grep $project_name.jar|awk -F '[ ]+' '{print $2}')
echo $pid
if [ -z $pid ]
then
  echo $(date +%F_%T) $project_name.jar "failed to start" >> $app_path/failure.log
  exit 1
else
  echo $(date +%F_%T) $project_name.jar PID:$pid "started successfully" >> $app_path/startup.log
fi
```

注意：之前使用的是以下脚本，但会报错：`Exec exit status not zero. Status [-1]`

*手动执行脚本是没问题的，其他环境也没问题，单独在 Jenkins 本地构建也没问题，偏在 Publish Over SSH 部署远程就会有问题*

==问题原因==：`grep -v grep` 因为这样写会导致 SSH 当前的执行进程也被 kill 掉(因为只过滤 grep)，从而返回 -1，而非正常结束的 0

```bash
#!/bin/env bash

# 接收外部参数
project_name=$1

PID=`ps -ef | grep -w $project_name | grep -v grep | awk '{print $2}'`

if [ ! "$PID" ]
then
    echo $PID"进程不存在"
else
    echo "进程存在 杀死进程PID$PID"
    kill -9 $PID
fi

nohup java -jar /opt/app/$project_name.jar >/dev/null 2>&1 &

# 根据重启后是否有当前应用判断启动是否成功
pid=$(ps -ef | grep java| grep $project_name.jar | awk -F '[ ]+' '{print $2}')
echo $pid
if [ -z $pid ]
then
  echo "启动失败"
  exit 1
else
  echo $project_name.jar :  $pid  "启动成功"
fi

```

#### 3.2.8 邮件服务

> Jenkins 默认是有一个邮件通知，但只是纯文本，太过于简单并无法定制化，下述为邮件增加版
>
> 其中以 `Default` 开头的名称，都可在 Job 配置中作变量使用

1. 安装 `Email Extension、Email Extension Template` 插件，配置默认发件箱与邮件类型

- SMTP Server：发件邮箱 smtp 服务器
- SMTP Port：smtp 服务器端口号
- Credentials：使用凭证方式，密码注意是否为**STMP授权码**
- Use SSL：SSL 加密，根据上面端口号勾选
- Default user e-mail suffix：默认发件邮箱后缀名
- Charset：字符类型 UTF-8
- Default Content Type：邮件内容类型，选 HTML(text/html)

![image-20220525102855202](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220525102855202.png)

****

**创建邮箱账密凭证信息**

![image-20220525103148083](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220525103148083.png)

2. 设置默认收件、邮件标题和邮件内容

- Default Recipients：默认收件邮箱，多人以逗号进行分割
- Default Subject：默认邮件标题(所有 Job 通用)
- Default Content：默认邮件内容(所有 Job 通用)

![image-20220525141021088](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220525141021088.png)

3. 设置触发类型与重复发件箱配置

> 触发类型：选得是 Always，可自行定制

![image-20220525141649805](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220525141649805.png)

重复发件箱配置与上面的 SMTP 配置一致即可，

![image-20220525141816928](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220525141816928.png)

注：配置完成后可通过发送测试邮件查看是否配置正确

4. 流水线增加 `post` 动作，构建后即发送邮件

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
        
        // 编译打包
        stage('Code Package') {
            steps {
                // 编译公共子工程
                //sh "mvn -f ruoyi-common clean install"
                // 编译并打包指定服务工程
                sh "mvn -f ${project_name} clean package -Dmaven.test.skip=true"
            }
        }
        
        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
        
        // 上线发布
        stage('Deploy Server') {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'debug', 
                // 清除远程目录内文件
                transfers: [sshTransfer(cleanRemote: true, 
                excludes: '', execCommand: "sh /home/yuen/deploy.sh $project_name", execTimeout: 120000, flatten: false, 
                makeEmptyDirs: false, 
                noDefaultExcludes: false, 
                patternSeparator: '[, ]+', 
                remoteDirectory: 'opt/app', remoteDirectorySDF: false, 
                removePrefix: '${project_name}/target', 
                sourceFiles: '${project_name}/target/*.jar')], 
                usePromotionTimestamp: false, 
                useWorkspaceInPromotion: false, 
                verbose: false)])
            }
        }

    }
    
    // 构建后动作:发送邮件
    post {
        always {
            emailext body:
            """
            <p>作业: <b>\'${env.JOB_NAME}:${env.BUILD_NUMBER})\' </b></p>
            <p>查看日志: "<a href="${env.BUILD_URL}"> ${env.JOB_NAME}:${env.BUILD_NUMBER}/console </a>"</p>
            <p><i>(构建构建日志请查看附件.)</i></p>
            """,
            compressLog: true,
            attachLog: true,
            subject: "作业 \'${env. JOB_NAME}:${env.BUILD_NUMBER}\' - 状态: ${currentBuild.result?:'SUCCESS'}", 
            to: '228003666@qq.com'
        }
    }     
}
```

emailext 步骤常用的参数如下：

- subject: String 类型，邮件主题；
- body: String 类型，邮件内容；
- attachLog: （可选）布尔类型，是否将构建日志以附件形式发送；
- attachmentsPattern: (可选) String 类型，需要发送的附件路径，Ant 风格路径表达式；
- compressLog: (可选) 布尔类型，是否压缩日志；
- from: (可选) String 类型，发件人邮箱；
- to: (可选) String 类型，收件人邮箱；
- recipientProviders: (可选) LIST 类型，收件人列表；
- replyTo: (可选) 回复邮箱；

![image-20220527135658455](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220527135658455.png)

5. 也可通过判断条件方式进行定制发送过程及内容，更多设置可参考`流水线语句--步骤参考--emailext:Extended Email`

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
        
        // 编译打包
        stage('Code Package') {
            steps {
                // 编译公共子工程
                //sh "mvn -f ruoyi-common clean install"
                // 编译并打包指定服务工程
                sh "mvn -f ${project_name} clean package -Dmaven.test.skip=true"
            }
        }
        
        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
        
        // 上线发布
        stage('Deploy Server') {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'debug', 
                // 清除远程目录内文件
                transfers: [sshTransfer(cleanRemote: true, 
                excludes: '', execCommand: "sh /home/yuen/deploy.sh $project_name", execTimeout: 120000, flatten: false, 
                makeEmptyDirs: false, 
                noDefaultExcludes: false, 
                patternSeparator: '[, ]+', 
                remoteDirectory: 'opt/app', remoteDirectorySDF: false, 
                removePrefix: '${project_name}/target', 
                sourceFiles: '${project_name}/target/*.jar')], 
                usePromotionTimestamp: false, 
                useWorkspaceInPromotion: false, 
                verbose: false)])
            }
        }

    }
    
    // 构建后动作:发送邮件
    post {
        success {
            emailext (
                subject: "'${env.JOB_NAME} [${env.BUILD_NUMBER}]' 更新正常",
                body: """
                <p>详情：</p>
                <p>SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'</p>
                <p>状态：${env.JOB_NAME} Jenkins 更新运行正常</p>
                <p>URL: ${env.BUILD_URL}</p>
                <p>项目名称: ${env.JOB_NAME}</p>
                <p>项目更新进度: ${env.BUILD_NUMBER}</p>
                """,
                to: '228003666@qq.com',
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
        failure {
            emailext (
                subject: "'${env.JOB_NAME} [${env.BUILD_NUMBER}]' 更新失败",
                body: """
                <p>详情：</p>
                <p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'</p>
                <p>状态：${env.JOB_NAME} Jenkins 运行失败</p>
                <p>URL: ${env.BUILD_URL}</p>
                <p>项目名称: ${env.JOB_NAME}</p>
                <p>项目更新进度: ${env.BUILD_NUMBER}</p>
                """,
                to: '228003666@qq.com',
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }     
}
```

![image-20220527150828199](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220527150828199.png)

6. **自定义模版**，为了更方便的管理维护邮件报告，可以通过统一模版来定制，免去维护每个 Job 任务

> 模版的应用可以在项目的根目录创建 `email.html`，再 Job 流水线上调用，或在 Jenkins 根目录下创建 `/root/.jenkins/email-templates` 文件夹，也是在 Job 流水线上调用即可

- 项目根目录创建 `email.html`，在 Job 流水线的 `body` 调用模版内容

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
        
        // 编译打包
        stage('Code Package') {
            steps {
                nodejs (nodeJSInstallationName: 'NodeJS 16', configId: 'fa93c439-03c8-43cf-ae06-65c59427cbfd') {
                    // npm编译安装
                    sh '''
                    cd ${project_name}
                    rm -rf node_modules package-lock.json && npm install
                    npm cache clean --force
                    npm install -g npm@8.11.0
                    npm run build:prod --report
                    '''
                }
            }
        }

        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }  
        
    }
    post {
        always {
            emailext(
                subject: '构建通知：${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
                body: '${FILE,path="email.html"}',
                to: '228003666@qq.com'
            )
        }
    }

}
```

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>
</head>

<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"
      offset="0">
<table width="95%" cellpadding="0" cellspacing="0"
       style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">
    <tr>
        <td>(本邮件是程序自动下发的，请勿回复！)</td>
    </tr>
    <tr>
        <td><h2>
            <font color="#0000FF">构建结果 - ${BUILD_STATUS}</font>
        </h2></td>
    </tr>
    <tr>
        <td><br />
            <b><font color="#0B610B">构建信息</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td>
            <ul>
                <li>项目名称&nbsp;：&nbsp;${PROJECT_NAME}</li>
                <li>构建编号&nbsp;：&nbsp;第${BUILD_NUMBER}次构建</li>
                <li>触发原因：&nbsp;${CAUSE}</li>
                <li>构建日志：&nbsp;<a href="${BUILD_URL}console">${BUILD_URL}console</a></li>
                <li>构建&nbsp;&nbsp;Url&nbsp;：&nbsp;<a href="${BUILD_URL}">${BUILD_URL}</a></li>
                <li>工作目录&nbsp;：&nbsp;<a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>
                <li>项目&nbsp;&nbsp;Url&nbsp;：&nbsp;<a href="${PROJECT_URL}">${PROJECT_URL}</a></li>
            </ul>
        </td>
    </tr>
    <tr>
        <td><b><font color="#0B610B">Changes Since Last
            Successful Build:</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td>
            <ul>
                <li>历史变更记录 : <a href="${PROJECT_URL}changes">${PROJECT_URL}changes</a></li>
            </ul> ${CHANGES_SINCE_LAST_SUCCESS,reverse=true, format="Changes for Build #%n:<br />%c<br />",showPaths=true,changesFormat="<pre>[%a]<br />%m</pre>",pathFormat="&nbsp;&nbsp;&nbsp;&nbsp;%p"}
        </td>
    </tr>
    <tr>
        <td><b>Failed Test Results</b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td><pre
                style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">$FAILED_TESTS</pre>
            <br /></td>
    </tr>
    <tr>
        <td><b><font color="#0B610B">构建日志 (最后 100行):</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td><textarea cols="80" rows="30" readonly="readonly"
                      style="font-family: Courier New">${BUILD_LOG, maxLines=100}</textarea>
        </td>
    </tr>
</table>
</body>
</html>
```

![image-20220528152841091](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220528152841091.png)

- Jenkins 根目录创建统一模版目录 `email-templates`，再创建官方 [**groovy-html.template**](https://github.com/jenkinsci/email-ext-plugin/blob/master/src/main/resources/hudson/plugins/emailext/templates/groovy-html.template)


```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }

        // 编译打包
        stage('Code Package') {
            steps {
                nodejs (nodeJSInstallationName: 'NodeJS 16', configId: 'fa93c439-03c8-43cf-ae06-65c59427cbfd') {
                    // npm编译安装
                    sh '''
                    cd ${project_name}
                    rm -rf node_modules package-lock.json && npm install
                    npm cache clean --force
                    npm install -g npm@8.11.0
                    npm run build:prod --report
                    '''
                }
            }
        }

        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
    }
    
    // 构建后邮件通知
    post {
        always {
            emailext(
                subject: '构建通知：${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
                // 调用指定的模版
                body: '''${SCRIPT,template="groovy-html.template"}''',
                to: '228003666@qq.com'
            )
        }
    }
}
```

```html
<STYLE>
  BODY, TABLE, TD, TH, P {
    font-family: Calibri, Verdana, Helvetica, sans serif;
    font-size: 12px;
    color: black;
  }
  .console {
    font-family: Courier New;
  }
  .filesChanged {
    width: 10%;
    padding-left: 10px;
  }
  .section {
    width: 100%;
    border: thin black dotted;
  }
  .td-title-main {
    color: white;
    font-size: 200%;
    padding-left: 5px;
    font-weight: bold;
  }
  .td-title {
    color: white;
    font-size: 120%;
    font-weight: bold;
    padding-left: 5px;
    text-transform: uppercase;
  }
  .td-title-tests {
    font-weight: bold;
    font-size: 120%;
  }
  .td-header-maven-module {
    font-weight: bold;
    font-size: 120%;    
  }
  .td-maven-artifact {
    padding-left: 5px;
  }
  .tr-title {
    background-color: <%= (build.result == null || build.result.toString() == 'SUCCESS') ? '#27AE60' : build.result.toString() == 'FAILURE' ? '#E74C3C' : '#f4e242' %>;
  }
  .test {
    padding-left: 20px;
  }
  .test-fixed {
    color: #27AE60;
  }
  .test-failed {
    color: #E74C3C;
  }
</STYLE>
<BODY>
  <!-- BUILD RESULT -->
  <table class="section">
    <tr class="tr-title">
      <td class="td-title-main" colspan=2>
        BUILD ${build.result ?: 'COMPLETED'}
      </td>
    </tr>
    <tr>
      <td>URL:</td>
      <td><A href="${rooturl}${build.url}">${rooturl}${build.url}</A></td>
    </tr>
    <tr>
      <td>Project:</td>
      <td>${project.name}</td>
    </tr>
    <tr>
      <td>Date:</td>
      <td>${it.timestampString}</td>
    </tr>
    <tr>
      <td>Duration:</td>
      <td>${build.durationString}</td>
    </tr>
    <tr>
      <td>Cause:</td>
      <td><% build.causes.each() { cause -> %> ${cause.shortDescription} <%  } %></td>
    </tr>
  </table>
  <br/>

  <!-- CHANGE SET -->
  <%
  def changeSets = build.changeSets
  if(changeSets != null) {
    def hadChanges = false %>
  <table class="section">
    <tr class="tr-title">
      <td class="td-title" colspan="2">CHANGES</td>
    </tr>
    <% changeSets.each() { 
      cs_list -> cs_list.each() { 
        cs -> hadChanges = true %>
    <tr>
      <td>
        Revision
        <%= cs.metaClass.hasProperty('commitId') ? cs.commitId : cs.metaClass.hasProperty('revision') ? cs.revision : cs.metaClass.hasProperty('changeNumber') ? cs.changeNumber : "" %>
        by <B><%= cs.author %></B>
      </td>
      <td>${cs.msgAnnotated}</td>
    </tr>
        <% cs.affectedFiles.each() {
          p -> %>
    <tr>
      <td class="filesChanged">${p.editType.name}</td>
      <td>${p.path}</td>
    </tr>
        <% }
      }
    }
    if ( !hadChanges ) { %>
    <tr>
      <td colspan="2">No Changes</td>
    </tr>
    <% } %>
  </table>
  <br/>
  <% } %>

<!-- ARTIFACTS -->
  <% 
  def artifacts = build.artifacts
  if ( artifacts != null && artifacts.size() > 0 ) { %>
  <table class="section">
    <tr class="tr-title">
      <td class="td-title">BUILD ARTIFACTS</td>
    </tr>
    <% artifacts.each() {
      f -> %>
      <tr>
        <td>
          <a href="${rooturl}${build.url}artifact/${f}">${f}</a>
      </td>
    </tr>
    <% } %>
  </table>
  <br/>
  <% } %>

<!-- MAVEN ARTIFACTS -->
  <%
  try {
    def mbuilds = build.moduleBuilds
    if ( mbuilds != null ) { %>
  <table class="section">
    <tr class="tr-title">
      <td class="td-title">BUILD ARTIFACTS</td>
    </tr>
      <%
      try {
        mbuilds.each() {
          m -> %>
    <tr>
      <td class="td-header-maven-module">${m.key.displayName}</td>
    </tr>
          <%
          m.value.each() { 
            mvnbld -> def artifactz = mvnbld.artifacts
            if ( artifactz != null && artifactz.size() > 0) { %>
    <tr>
      <td class="td-maven-artifact">
              <% artifactz.each() {
                f -> %>
        <a href="${rooturl}${mvnbld.url}artifact/${f}">${f}</a><br/>
              <% } %>
      </td>
    </tr>
            <% }
          }
        }
      } catch(e) {
        // we don't do anything
      } %>
  </table>
  <br/>
    <% }
  } catch(e) {
    // we don't do anything
  } %>

<!-- JUnit TEMPLATE -->

  <%
  def junitResultList = it.JUnitTestResult
  try {
    def cucumberTestResultAction = it.getAction("org.jenkinsci.plugins.cucumber.jsontestsupport.CucumberTestResultAction")
    junitResultList.add( cucumberTestResultAction.getResult() )
  } catch(e) {
    //cucumberTestResultAction not exist in this build
  }
  if ( junitResultList.size() > 0 ) { %>
  <table class="section">
    <tr class="tr-title">
      <td class="td-title" colspan="5">${junitResultList.first().displayName}</td>
    </tr>
    <tr>
        <td class="td-title-tests">Name</td>
        <td class="td-title-tests">Failed</td>
        <td class="td-title-tests">Passed</td>
        <td class="td-title-tests">Skipped</td>
        <td class="td-title-tests">Total</td>
      </tr>
    <% junitResultList.each {
      junitResult -> junitResult.getChildren().each {
        packageResult -> %>
    <tr>
      <td>${packageResult.getName()}</td>
      <td>${packageResult.getFailCount()}</td>
      <td>${packageResult.getPassCount()}</td>
      <td>${packageResult.getSkipCount()}</td>
      <td>${packageResult.getPassCount() + packageResult.getFailCount() + packageResult.getSkipCount()}</td>
    </tr>
    <% packageResult.getPassedTests().findAll({it.getStatus().toString() == "FIXED";}).each{
        test -> %>
            <tr>
              <td class="test test-fixed" colspan="5">
                ${test.getFullName()} ${test.getStatus()}
              </td>
            </tr>
        <% } %>
        <% packageResult.getFailedTests().sort({a,b -> a.getAge() <=> b.getAge()}).each{
          failed_test -> %>
    <tr>
      <td class="test test-failed" colspan="5">
        ${failed_test.getFullName()} (Age: ${failed_test.getAge()})
      </td>
    </tr>
        <% }
      }
    } %>
  </table>
  <br/>
  <% } %>

<!-- CONSOLE OUTPUT -->
  <%
  if ( build.result == hudson.model.Result.FAILURE ) { %>
  <table class="section" cellpadding="0" cellspacing="0">
    <tr class="tr-title">
      <td class="td-title">CONSOLE OUTPUT</td>
    </tr>
    <% 	build.getLog(100).each() {
      line -> %>
	  <tr>
      <td class="console">${org.apache.commons.lang.StringEscapeUtils.escapeHtml(line)}</td>
    </tr>
    <% } %>
  </table>
  <br/>
  <% } %>
</BODY>
```

![image-20220528155743921](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220528155743921.png)

小知识：此处虽然未配置邮件发送，但默认配置了 `always` 发送邮件，因此会采用默认 `Default Content`

> 默认邮件可修改`Default Content` 为 `${SCRIPT,template="groovy-html.template"}`，那就会按此模版进行发送

![image-20220528164952622](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220528164952622.png)

#### 3.2.9 前端服务

1. 安装 NodeJS 插件
2. 安装 NodeJS，并配置 NodeJS 环境变量，"Manage Jenkins" -> "Global Tool Configuration" -> 输入名字："NodeJS 16"

![image-20220528120406846](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220528120406846.png)

3. 配置 NPM 的源为国内淘宝源，"Manage Jenkins" -> "Managed files" -> "Add a new Config" -> 选择 "Npm config file" -> 提交

```
在 "Content" 处添加
registry =  http://registry.npmmirror.com
```

![image-20220528122533514](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220528122533514.png)

注意记录 ID-fa93c439-03c8-43cf-ae06-65c59427cbfd，后面在 Build 阶段会指定该 ID

4. 构建步骤与后端服务一样，因为项目依赖问题，将前后服务区分不同 Jenkinsfile 文件，或自行拆分项目

![image-20220528173718769](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220528173718769.png)

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }

        // 编译打包
        stage('Code Package') {
            steps {
                nodejs (nodeJSInstallationName: 'NodeJS 16', configId: 'fa93c439-03c8-43cf-ae06-65c59427cbfd') {
                    // npm编译安装
                    sh '''
                    cd ${project_name}
                    rm -rf node_modules package-lock.json && npm install
                    npx browserslist@latest --update-db
                    npm cache clean --force
                    npm install -g npm@8.11.0
                    npm run build:prod --report
                    '''
                }
            }
        }

        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
        
        // 上线发布
        stage('Deploy Server') {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'debug', 
                transfers: [sshTransfer(cleanRemote: true, 
                excludes: '', execCommand: '', execTimeout: 120000, flatten: false, 
                makeEmptyDirs: false, 
                noDefaultExcludes: false, 
                patternSeparator: '[, ]+', 
                remoteDirectory: '/opt/nginx/html', 
                remoteDirectorySDF: false, 
                removePrefix: '${project_name}/dist', 
                sourceFiles: '${project_name}/dist/*')], 
                usePromotionTimestamp: false, 
                useWorkspaceInPromotion: false, 
                verbose: false)])
               }
            }
        }
    
    // 构建后发送邮件
    post {
        always {
            emailext(
                subject: '构建通知：${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
                body: '${FILE,path="email.html"}',
                //body: '''${SCRIPT,template="groovy-html.template"}''',
                to: '228003666@qq.com'
            )
        }
    }
}
```

### 3.3 容器镜像

> 以上教程为 `SpringBoot` 前后端分离单机发版方式，后面将讲解代码生成容器镜像，并自动化发布至镜像仓库、指定生产服务器等操作

**操作流程**

- 开发人员修改代码提交至 Gitlab 代码仓库
- 测试人员在 Jenkins 手动操作构建，通过从 Gitlab 中拉取项目代码，编译并构建成 Docker 镜像，之后上传到 Harbor 私有仓库并发布到指定测试服务器 (可通过代码提交进行触发构建)
- 项目经理在 Jenkins 手动操作构建，Jenkins 通过发送 SSH 远程命令执行脚本文件，让生产服务器到 Harbor 私有仓库拉取指定版本的镜像到本地，然后创建容器供其访问

#### 3.3.1 配置环境

1. 安装 Docker（前面安装 Harbor 时已演示过，请自行查阅，此略过）

   > 为了后期的镜像构建，需要 Docker 环境，现将在 Jenkins & Client 两台机器上进行安装
   >
   > - Jenkins 用于构建镜像并上传 Harbor 镜像仓库
   > - Client 用于演示镜像下载及容器创建运行

2. 配置 Harbor 访问证书（因搭建的是 HTTPS-Harbor，需要操作系统级别的信任证书）

```bash
$ scp -P 44  /etc/docker/certs.d/harbor.yuikuen.top/ca.crt root@188.188.4.44:/etc/pki/ca-trust/source/anchors/
$ scp -P 22  /etc/docker/certs.d/harbor.yuikuen.top/ca.crt root@188.188.4.222:/etc/pki/ca-trust/source/anchors/
$ update-ca-trust

# 注意要重启Docker服务方可登录 
$ systemctl daemon-reload && systemctl restart docker && systemctl enable docker

$ docker login -u admin -p Harbor12345 harbor.yuikuen.top
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

确保容器镜像服务能正常并能正常访问 Harbor 即可进行下一步操作

#### 3.3.2 配置插件

> 上面已演示了通过 Maven 进行编译打包，会生成对应的 Jar 文件，而后通过 `dockerfile-maven-plugin` 插件可以在项目构建的时候自动生成镜像，并将生成的镜像 push 到指定的镜像库，或在指定服务器上创建基于其镜像的容器提供相应的服务

1. 项目代码 `pom文件` 配置 `dockerfile-maven-plugin` 插件

```xml
        <plugins>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>dockerfile-maven-plugin</artifactId>
                <version>1.4.13</version>
                <executions>
                    <execution>
                        <id>default</id>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <repository>${project.artifactId}</repository>
                    <tag>${project.version}</tag>
                    <!-->buildArgs元素指定传递给Dockerfile的参数<-->
                    <buildArgs>
                        <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
                    </buildArgs>
                </configuration>
            </plugin>
        </plugins>
```

注解：

- `execution` 指定了该插件 `build` 目标使用的默认绑定至 package 阶段
- `repository` 是生成的镜像的 repository 信息
- `tag` 指定镜像的 tag，这里使用的是 Maven 模块的版本号
- `buildArgs` 元素指定了传递给 Dockerfile 的参数，比如上面 Dockerfile 中的 JAR_FILE.
- `${project.build.finalName}.jar` 是 jar 包路径，这里使用了最终生成的 jar 包的文件名

**以 `ruoyi-gateway` 为例，增加插件模块**（tag 值通过后期的构建次数来获取，此处可注释）

![image-20220529142855505](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220529142855505.png)

### 3.3.3 构建文件

> 在子项目的根目录创建 Dockerfile 构建文件，根据项目内容进行可进行定制（下述为简单例子）

```dockerfile
# 获取ARG值，复制jar文件并重名，指定端口运行
FROM openjdk:8-jdk-alpine
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app.jar"]
```

#### 3.3.4 项目构建

> 提醒：项目多为后端服务，因此将 `Jenkinsfile` 放置父项目根目录，前端服务部署注意层级关系即可
>
> 懂 SpringBoot/SpringCloud 的可自行拆分项目，便于区分项目也行，在此不作拆解

1. 修改 `Jenkinsfile` 文件，重新指定编译打包和镜像构建

```groovy
pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
        
        // 编译打包
        stage('Code Package') {
            steps {
                // 编译公共子工程
                //sh "mvn -f ruoyi-common clean install"
                // 编译并打包指定服务工程
                //sh "mvn -f ${project_name} clean package -Dmaven.test.skip=true"
                // 编译方式改成dockerfile:build
                sh "mvn -f ${project_name} clean package dockerfile:build"
            }
        }
        
        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }

    }     
}
```

2. 镜像构建成功后，Jenkins 服务器会生成子项目的镜像，下一步按照 Docker 操作，重新打标签并登录 Harbor 上传镜像

```bash
$ docker images
REPOSITORY                                 TAG            IMAGE ID       CREATED       SIZE
ruoyi-gateway                              latest         c73dd4f6effb   5 days ago    201MB
openjdk                                    8-jdk-alpine   a3562aa0b991   3 years ago   105MB
```

3. Jenkinsfile 增加流水线上传镜像步骤，注意此处 def 定义了 Harbor 仓库作变量，需要提前创建 `Secret`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220529151635721.png)

**流水线语法：添加绑定 Harbor 的账密信息**

![image-20220531102933210](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220531102933210.png)

```groovy
// Harbor仓库地址
def harbor_url = "harbor.yuikuen.top"
// Harbor项目名称
def harbor_project_name = "library"
// Harbor登录凭证ID
def harbor_auth = "2c07d30c-cbe6-4a21-b908-71c076a941c0"

pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }

        // 编译打包
        stage('Code Package & Build') {
            steps {
                // 编译并打包指定服务工程，构建本地镜像
                sh "mvn -f ${project_name} clean package dockerfile:build"
            }
        }
        
        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
        
        // 推送镜像
        stage('Tag & Pull Image') {
            steps {
                // 镜像重新打标签
                sh "docker tag ${project_name}:latest ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
                // 登录Harbor,并上传镜像（Harbor账密信息）
                withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'harbor_passwd', usernameVariable: 'harbor_name')]) {
                    sh "docker login -u ${harbor_name} -p ${harbor_passwd} ${harbor_url}"
                    sh "docker push ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
                }
                // 删除本地镜像
                sh "docker rmi -f ${project_name}:latest"
                sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
            }
        }
    }     
}
```

**检查构建结果：Jenkins 本地镜像、Harbor 镜像**

```bash
[Sun May 29 root@jenkins ~]# docker images
REPOSITORY   TAG            IMAGE ID       CREATED       SIZE
openjdk      8-jdk-alpine   a3562aa0b991   3 years ago   105MB
```

通过查看 Jenkins 服务器镜像信息和构建日志，已按编排的步骤进行打标签、上传、删除等操作

![image-20220529153937490](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220529153937490.png)

在 Harbor 指定仓库可以查看到提交的镜像

![image-20220529154259029](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220529154259029.png)

#### 3.3.5 远端部署

> 操作步骤与 Jar 运行一样，发送远程命令到指定服务器执行脚本：停止|删除本地运行的容器及镜像，重新拉取部署

1. Jenkinsfile 增加远程上线步骤，注意此次不必发送远程文件

```groovy
// Harbor仓库地址
def harbor_url = "harbor.yuikuen.top"
// Harbor项目名称
def harbor_project_name = "library"
// Harbor登录凭证ID
def harbor_auth = "2c07d30c-cbe6-4a21-b908-71c076a941c0"

pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }

        // 编译打包
        stage('Code Package & Build') {
            steps {
                // 编译并打包指定服务工程，构建本地镜像
                sh "mvn -f ${project_name} clean package dockerfile:build"
            }
        }
        
        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
        
        // 推送镜像
        stage('Tag & Pull Image') {
            steps {
                // 镜像重新打标签
                sh "docker tag ${project_name}:latest ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
                // 登录Harbor,并上传镜像
                withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'harbor_passwd', usernameVariable: 'harbor_name')]) {
                    sh "docker login -u ${harbor_name} -p ${harbor_passwd} ${harbor_url}"
                    sh "docker push ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
                }
                // 删除本地镜像
                sh "docker rmi -f ${project_name}:latest"
                sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
            }
        }
        
        // 远程部署
        stage('Deploy Server') {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'debug', 
                transfers: [sshTransfer(cleanRemote: false, 
                excludes: '', 
                // 注意此处变量的值都是为了让远程服务器获取使用
                execCommand: "sh /opt/spring/deploy.sh $harbor_url $harbor_project_name $project_name ${BUILD_ID}", 
                execTimeout: 120000, flatten: false, 
                makeEmptyDirs: false, 
                noDefaultExcludes: false, 
                patternSeparator: '[, ]+', 
                remoteDirectory: '', 
                remoteDirectorySDF: false, 
                removePrefix: '', 
                sourceFiles: '')], 
                usePromotionTimestamp: false, 
                useWorkspaceInPromotion: false, 
                verbose: false)])
            }
        }
    }     
}
```

2. 远程服务器创建脚本文件 `/opt/spring/deploy.sh`

```shell
#! /bin/bash

# 接收外部参数
harbor_url=$1
harbor_project_name=$2
project_name=$3
tag=$4

app_path=/opt/spring
imageName=$harbor_url/$harbor_project_name/$project_name:$tag
echo "$imageName"

containerId=`docker ps -a | grep -w $project_name | awk '{print $1}'`
if [ "$containerId" != "" ] ; then
    docker stop $containerId
    docker rm $containerId
    echo $(date +%F_%T) "成功删除容器" >> $app_path/success.log
else
    echo $(date +%F_%T) $containerId "is null" >> $app_path/failure.log
fi

imageId=`docker images | grep -w $project_name | awk '{print $3}'`
if [ "$imageId" != "" ] ; then
    docker rmi -f $imageId
    echo $(date +%F_%T) "成功删除镜像" >> $app_path/success.log
else
    echo $(date +%F_%T) $imageId "is null" >> $app_path/failure.log
fi

docker run -itd --net host --name=$project_name --restart=always $imageName

echo $(date +%F_%T) "容器启动成功" >> $app_path/success.log
```

3. 构建成功后检查远端服务器的变化是否如构建步骤及脚本执行

```bash
[Sun May 29 root@debug spring]# docker images
REPOSITORY                                 TAG       IMAGE ID       CREATED         SIZE
harbor.yuikuen.top/library/ruoyi-gateway   174       7c1c6d9bcd47   8 minutes ago   201MB
[Sun May 29 root@debug spring]# docker ps
CONTAINER ID   IMAGE                                          COMMAND                CREATED         STATUS         PORTS     NAMES
904dba82276e   harbor.yuikuen.top/library/ruoyi-gateway:174   "java -jar /app.jar"   8 minutes ago   Up 8 minutes             ruoyi-gateway

$ cat /opt/spring/success.log 
2022-05-29_16:29:20 成功删除容器
2022-05-29_16:29:20 成功删除镜像
2022-05-29_16:29:28 容器启动成功
2022-05-29_16:46:03 成功删除容器
2022-05-29_16:46:03 成功删除镜像
2022-05-29_16:46:10 容器启动成功
2022-05-29_16:51:47 成功删除容器
2022-05-29_16:51:47 成功删除镜像
2022-05-29_16:51:54 容器启动成功
2022-05-29_16:56:47 成功删除容器
2022-05-29_16:56:48 成功删除镜像
2022-05-29_16:56:54 容器启动成功
```

![image-20220529163247953](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220529163247953.png)

**查看远程服务器，可以看到旧的镜像文件和容器已作变更**

```bash
[Sun May 29 root@debug spring]# docker images && docker ps -a
REPOSITORY                                 TAG       IMAGE ID       CREATED              SIZE
harbor.yuikuen.top/library/ruoyi-gateway   175       31e8b3c28f19   About a minute ago   201MB
CONTAINER ID   IMAGE                                          COMMAND                CREATED              STATUS              PORTS     NAMES
eb6d91a1fb4e   harbor.yuikuen.top/library/ruoyi-gateway:175   "java -jar /app.jar"   About a minute ago   Up About a minute             ruoyi-gateway

# 对应的Java服务也正常启动
$ ps -ef|grep java
nacos      1323      1  1 11:23 ?        00:05:44 /opt/java/bin/java -Djava.ext.dirs=/opt/java/jre/lib/ext:/opt/java/lib/ext -Xms512m -Xmx512m -Xmn256m -Dnacos.standalone=true -Dnacos.member.list= -Xloggc:/opt/nacos/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/opt/nacos/plugins/health,/opt/nacos/plugins/cmdb -Dnacos.home=/opt/nacos -jar /opt/nacos/target/nacos-server.jar --spring.config.additional-location=file:/opt/nacos/conf/ --logging.config=/opt/nacos/conf/nacos-logback.xml --server.max-http-header-size=524288 nacos.nacos
seata      1982   1981  0 11:24 ?        00:02:15 /opt/java//bin/java -server -Xmx2048m -Xms2048m -Xmn1024m -Xss512k -XX:SurvivorRatio=10 -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=1024m -XX:-OmitStackTraceInFastThrow -XX:-UseAdaptiveSizePolicy -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/seata/logs/java_heapdump.hprof -XX:+DisableExplicitGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=75 -Xloggc:/opt/seata/logs/seata_gc.log -verbose:gc -Dio.netty.leakDetectionLevel=advanced -classpath /opt/seata/conf:/opt/seata/lib/* -Dapp.name=seata-server -Dapp.pid=1982 -Dapp.repo=/opt/seata/lib -Dapp.home=/opt/seata -Dbasedir=/opt/seata io.seata.server.Server -p 8091 -h 188.188.4.44
root      23665  23646 25 16:29 pts/0    00:01:12 java -jar /app.jar
root      24316  22782  0 16:34 pts/0    00:00:00 grep --color=auto java
```

#### 3.3.6 邮件服务

> 邮件服务同样采用模版方式直接调用，可参考之前的配置，修改 Jenkinsfile 增加步骤

```groovy
// Harbor仓库地址
def harbor_url = "harbor.yuikuen.top"
// Harbor项目名称
def harbor_project_name = "library"
// Harbor登录凭证ID
def harbor_auth = "2c07d30c-cbe6-4a21-b908-71c076a941c0"

pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }

        // 编译打包
        stage('Code Package & Build') {
            steps {
                // 编译并打包指定服务工程，构建本地镜像
                sh "mvn -f ${project_name} clean package dockerfile:build"
            }
        }
        
        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
        
        // 推送镜像
        stage('Tag & Pull Image') {
            steps {
                // 镜像重新打标签
                sh "docker tag ${project_name}:latest ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
                // 登录Harbor,并上传镜像
                withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'harbor_passwd', usernameVariable: 'harbor_name')]) {
                    sh "docker login -u ${harbor_name} -p ${harbor_passwd} ${harbor_url}"
                    sh "docker push ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
                }
                // 删除本地镜像
                sh "docker rmi -f ${project_name}:latest"
                sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
            }
        }
        
        // 远程部署
        stage('Deploy Server') {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'debug', 
                transfers: [sshTransfer(cleanRemote: false, 
                excludes: '', 
                execCommand: "sh /opt/spring/deploy.sh $harbor_url $harbor_project_name $project_name ${BUILD_ID}", 
                execTimeout: 120000, flatten: false, 
                makeEmptyDirs: false, 
                noDefaultExcludes: false, 
                patternSeparator: '[, ]+', 
                remoteDirectory: '', 
                remoteDirectorySDF: false, 
                removePrefix: '', 
                sourceFiles: '')], 
                usePromotionTimestamp: false, 
                useWorkspaceInPromotion: false, 
                verbose: false)])
            }
        }

        // 邮件提醒
        stage('Send Mail') {
            steps {
                echo '构建后邮件发送'
            }
            post {
                always {
                    emailext(
                        subject: '构建通知：${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
                        body: '${FILE,path="email.html"}',
                        to: '228003666@qq.com',
                        compressLog: true,
                        attachLog: true
                    )
                }
            }   
        }
    }
}
```

![image-20220529165820678](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220529165820678.png)

![image-20220529165920677](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220529165920677.png)

**小知识补充：如果构建失败次数过多，可能会产生很多 `<none>` 的无用镜像，建议在 Jenkins 创建定期清理的脚本**

```bash
# 定时每天夜晚1点删除3天(72h)之前的images
$ crontab -e
$ 0 1 * * * docker image prune -a --force --filter "until=72h"
$ systemctl restart crond.service
```

#### 3.3.7 前端服务

> 前端容器构建方式与之前的操作步骤差不多，只是增加镜像构建、推送的动作，并且远程部署也是采用 SSH 远程命令执行脚本

![image-20220601114259492](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220601114259492.png)

1. 根目录创建 `Dockerfile` 文件，定义镜像打包的过程

> 下述的路径是根据基础镜像的默认配置进行修改的，可自行定义

```dockerfile
# 基础镜像
FROM nginx:1.22.0-alpine-perl
# MAINTAINER YuiKuen
# 删除默认数据
RUN rm -rf /etc/nginx/conf.d/* /usr/share/nginx/html/*
# 复制文件到上述路径
COPY default.conf /etc/nginx/conf.d/default.conf
COPY dist/* /usr/share/nginx/html/
# 修改文件权限
RUN chmod o+rx /usr/share/nginx/html && sed -i '2c user root;' /etc/nginx/nginx.conf
```

2. 根目录创建 Nginx 配置文件 `default.conf`，参考原若依配置文件进行修改

```bash
#worker_processes  1;
#
#events {
#    worker_connections  1024;
#}
#
#http {
#    include       mime.types;
#    default_type  application/octet-stream;
#    sendfile        on;
#    keepalive_timeout  65;
#
    server {
        listen       80;
        server_name  localhost;

		location / {
            root   /usr/share/nginx/html;
			try_files $uri $uri/ /index.html;
            index  index.html index.htm;
        }
		
		location /prod-api/{
			proxy_set_header Host $http_host;
			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header REMOTE-HOST $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass http://188.188.4.44:8080/;
		}

       # error_page   500 502 503 504  /50x.html;
       # location = /50x.html {
       #     root   html;
       # }
    }
#}
# requirepass 123456
```

3. 修改 `Jenkinsfile` 文件，主要修改构建镜像的动作

```groovy
// Harbor仓库地址
def harbor_url = "harbor.yuikuen.top"
// Harbor项目名称
def harbor_project_name = "library"
// Harbor登录凭证ID
def harbor_auth = "2c07d30c-cbe6-4a21-b908-71c076a941c0"

pipeline {
    agent any

    stages {
        // 拉取代码
        stage('Clone Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "*/${branch}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins-id_rsa', url: 'git@gitlab.yuikuen.top:dev/RuoYi-Cloud.git']]])
            }
        }
        
        // 编译打包
        stage('Code Package') {
            steps {
                nodejs (nodeJSInstallationName: 'NodeJS 16', configId: 'fa93c439-03c8-43cf-ae06-65c59427cbfd') {
                    sh '''
                    cd ${project_name}
                    rm -rf node_modules package-lock.json && npm install
                    npx browserslist@latest --update-db
                    npm cache clean --force
                    npm install -g npm@8.11.0
                    npm run build:prod --report
                    '''
                }
            }
        }

        // 代码审查
        stage('Scan Code') {
            steps {
                script {
                    // 定义当前Jenkins的SonarQube工具
                    scannerHome = tool 'SonarQube-Scanner'
                }
                // 引用当明JenkinsSonarQube环境
                withSonarQubeEnv('SonarQube') {
                    sh """
                    cd ${project_name}
                    ${scannerHome}/bin/sonar-scanner
                    """
                }
            }
        }
        
        // 构建并推送镜像
        stage('Build & Pull Image') {
            steps {
                // 构建、重新打标签
                sh """
                cd ${project_name}
                docker build -t ${project_name} .
                docker tag ${project_name}:latest ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}
                """
                // 引上def变量，登录harbor并推送镜像
                withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'harbor_passwd', usernameVariable: 'harbor_name')]) {
                    sh "docker login -u ${harbor_name} -p  ${harbor_passwd} ${harbor_url}"
                    sh "docker push ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
                }
                // 删除本地镜像
                sh "docker rmi -f ${project_name}:latest"
                sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${project_name}:${BUILD_ID}"
            }
        }

        // 远程部署
        stage('Deploy Server') {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'debug', 
                transfers: [sshTransfer(cleanRemote: false, 
                excludes: '', 
                execCommand: "sh /opt/spring/deploy.sh $harbor_url $harbor_project_name $project_name ${BUILD_ID}", 
                execTimeout: 120000, flatten: false, 
                makeEmptyDirs: false, 
                noDefaultExcludes: false, 
                patternSeparator: '[, ]+', 
                remoteDirectory: '', 
                remoteDirectorySDF: false, 
                removePrefix: '', 
                sourceFiles: '')], 
                usePromotionTimestamp: false, 
                useWorkspaceInPromotion: false, 
                verbose: false)])
            }
        }

    }

    post {
        always {
            emailext(
                subject: '构建通知：${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
                body: '''${SCRIPT,template="groovy-html.template"}''',
                to: '228003666@qq.com',
                compressLog: true,
                attachLog: true
            )
        }
    }
}
```

脚本文件可沿用之前的远端部署的文件，为了方便区分项目可自行修改脚本

```bash
#! /bin/bash

# 接收外部参数
harbor_url=$1
harbor_project_name=$2
project_name=$3
tag=$4

app_path=/opt/spring
imageName=$harbor_url/$harbor_project_name/$project_name:$tag
echo "$imageName"

containerId=`docker ps -a | grep -w $project_name | awk '{print $1}'`
if [ "$containerId" != "" ] ; then
    docker stop $containerId
    docker rm $containerId
    echo $(date +%F_%T) $containerId "成功删除容器" >> $app_path/success.log
else
    echo $(date +%F_%T) $containerId "is null" >> $app_path/failure.log
fi

imageId=`docker images | grep -w $project_name | awk '{print $3}'`
if [ "$imageId" != "" ] ; then
    docker rmi -f $imageId
    echo $(date +%F_%T) $imageId "成功删除镜像" >> $app_path/success.log
else
    echo $(date +%F_%T) $imageId "is null" >> $app_path/failure.log
fi

docker run -itd --net host --name=$project_name --restart=always $imageName

echo $(date +%F_%T) $project_name "容器启动成功" >> $app_path/success.log
```
**通过输出的日志可以查看到对应项目的镜像&容器的变化**

```bash
$ cat success.log 
2022-06-01_10:56:20 d85dc159cc34 成功删除容器
2022-06-01_10:56:20 17625b659a97 成功删除镜像
2022-06-01_10:56:24 ruoyi-ui 容器启动成功
2022-06-01_14:32:25 fa5fb4ef218e 成功删除容器
2022-06-01_14:32:25 aa894bbaf698 成功删除镜像
2022-06-01_14:32:29 ruoyi-ui 容器启动成功
2022-06-01_14:39:51 df725a078b51 成功删除容器
2022-06-01_14:39:51 0381c982b6ee 成功删除镜像
2022-06-01_14:39:55 ruoyi-ui 容器启动成功

$ docker ps
CONTAINER ID   IMAGE                                          COMMAND                  CREATED         STATUS         PORTS     NAMES
9ee4eb3a92d0   harbor.yuikuen.top/library/ruoyi-ui:63         "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes             ruoyi-ui
f94d5c914e3f   harbor.yuikuen.top/library/ruoyi-gateway:178   "java -jar /app.jar"     2 days ago      Up 2 days                ruoyi-gateway
```

![image-20220601144744919](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220601144744919.png)
