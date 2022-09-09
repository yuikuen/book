# System Init

> 注意：做任何操作前，请提前做好对应的文件备份，以防配置错误导致系统崩溃；

1）关闭安全配置

```bash
$ /bin/cp /etc/selinux/config /etc/selinux/config.bak.$(date +%F_%T)
$ sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
$ systemctl disable --now firewalld
```

2）安装常用基础命令

```bash
$ yum -y install vim expect screen lrzsz tree openssl openssh-clients openssl-devel openssh-server telnet iftop iotop sysstat wget ntpdate dos2unix lsof net-tools mtr gcc gcc-c++ cmake zip unzip git sudo psmisc
```

3）配置阿里云源

```bash
$ mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak.$(date +%F_%T)
$ wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
$ wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
$ yum clean all && yum makecache
```

4）同步服务器时间

```bash
$ /usr/sbin/ntpdate ntp1.aliyun.com &> /dev/null &&  hwclock --systohc &> /dev/null
$ echo "*/5 * * * * /usr/sbin/ntpdate ntp1.aliyun.com &&  hwclock --systohc" >> /var/spool/cron/root
$ systemctl restart crond
```

5）修改样式，优化 history 记录

```bash
$ mkdir -p /var/log/command_history
$ vim /etc/profile
export PS1="[\e[1;34;40m\]\d \e[31;40m\]\u\e[32;40m\]@\e[35;40m\]\h \e[36;40m\]\W\[\e[0m\]]\\$ "
export HISTSIZE=100
export HISTTIMEFORMAT="<%F %T-$(who -u am i | awk '{print $NF}' | sed -e 's/[()]//g')-$(whoami)>"
export HISTORY_FILE=/var/log/command_history/`date '+%F'`.log
export PROMPT_COMMAND='{ date "+%F %T $(history 1 | { read x cmd; echo "$cmd !![USER:$USER FROM IP:$SSH_CLIENT PS:$SSH_TTY]"; })"; } >> $HISTORY_FILexport PROMPT_COMMAND='{ date "+%F %T $(history 1 | { read x cmd; echo "$cmd !![USER:$USER FROM IP:$SSH_CLIENT PS:$SSH_TTY]"; })"; } >> $HISTORY_FILE'E'
export TMOUT=300
$ source /etc/profile
```

6）调整用户级别的文件描述符数量及进程数量

```bash
$ /bin/cp /etc/security/limits.conf /etc/security/limits.conf.bak.$(date +%F_%T)
$ cat >> /etc/security/limits.conf << EOF
*   soft    nofile      65535
*   hard    nofile      65535
*   soft    nproc       65535
*   hard    nproc       65535
EOF

$ /bin/cp /etc/security/limits.d/20-nproc.conf  /etc/security/limits.d/20-nproc.conf.bak.$(date +%F_%T)
$ echo -e '*      soft    nproc     65535\nroot   soft    nproc     unlimited' > /etc/security/limits.d/20-nproc.conf
```

7）修改字符集

```bash
$ /bin/cp /etc/locale.conf /etc/locale.conf.bak.$(date +%F_%T)
$ echo 'LANG="en_US.UTF-8"' > /etc/locale.conf
$ source /etc/locale.conf
```

8）内核参数优化

```bash
$ /bin/cp /etc/sysctl.conf /etc/sysctl.conf.bak.$(date +%F_%T)
$ cat >> /etc/sysctl.conf <<EOF
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
EOF

$ sysctl -p &> /dev/null
```

9）限制远程连接及设置策略

```bash
$ /bin/cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F_%T)
$ sed -i 's/\#Port 22/Port 65522/' /etc/ssh/sshd_config
$ sed -i 's/\#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
$ sed -i 's/\#PermitEmptyPasswords no/PermitEmptyPasswords no/' /etc/ssh/sshd_config
$ sed -i 's/\#UseDNS yes/UseDNS no/' /etc/ssh/sshd_config
```