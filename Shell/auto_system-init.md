# Auto System-Init

> CentOS6~7 服务器初始化脚本

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