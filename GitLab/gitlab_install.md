# GitLab Install(RPM)

> Linux 快速安装，通过 rpm 源进行在线或离线安装

## 版本环境

- System：CentOS7.9.2009 Minimal
- GitLab：gitlab-ce-14.1.0

1）开放服务协议或直接禁用安全配置

```bash
$ firewall-cmd --add-service=ssh --permanent
$ firewall-cmd --add-service=http --permanent
$ firewalld-cmd --reload

$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service
```

2）下载依赖组件

```bash
$ yum -y install policycoreutils openssh-server openssh-clients postfix
$ systemctl enable --now sshd postfix
```

## 程序安装

> 安装过程分为在线和离线，都是通过
> [清华源-GitLab](https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/) 来进行安装

### 在线安装

1）配置清华源，查看现有版本并直接下载安装
```bash
$ cat /etc/yum.repos.d/gitlab-ce.repo
[gitlab-ce]
name=gitlab-ce
baseurl=http://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7
repo_gpgcheck=0
gpgcheck=0
enabled=1
gpgkey=https://packages.gitlab.com/gpg.key

$ yum list | grep gitlab-ce  
$ yum -y install gitlab-ce-14.1.0
```

### 离线安装

1）到清华源下载 rpm 源文件，直接解压安装

```bash
$ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-14.1.0-ce.0.el7.x86_64.rpm --no-check-certificate
$ rpm -ivh gitlab-ce-14.1.0-ce.0.el7.x86_64.rpm --force --nodeps
```

## 配置程序

> 默认的安装配置直接初始化就可以使用，其它设置可根据[官网配置](https://docs.gitlab.cn/jh/install/next_steps.html)

1）修改 `/etc/gitlab/gitlab.rd` 配置文件

```bash
## GitLab URL 
# 根据实际配置，IP或域名
# external_url 'http://gitlab.example.com'
external_url 'http://ip or hostname'
```

2）初始化服务，注意每次修改配置后需要执行方可生效

```bash
$ gitlab-ctl reconfigure
$ gitlab-ctl restart
```

3）初始管理员为 root，密码随机生成并保存在 `/etc/gitlab/initial_root_password`，有效期为 24 小时

```bash
Running handlers:
Running handlers complete
Chef Infra Client finished, 572/1520 resources updated in 04 minutes 16 seconds

Notes:
Default admin account has been configured with following details:
Username: root
Password: You didn't opt-in to print initial root password to STDOUT.
Password stored to /etc/gitlab/initial_root_password. This file will be cleaned up in first reconfigure run after 24 hours.

NOTE: Because these credentials might be present in your log files in plain text, it is highly recommended to reset the password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

gitlab Reconfigured!

$ cat /etc/gitlab/initial_root_password
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: d5WxYmXvXbEOfpzm/LZ6Bibmc+HzNTGpjNQU/3hdbh0=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

4）查看服务状态，并登录浏览器验证

```bash
$ gitlab-ctl status
down: alertmanager: 0s, normally up, want up; run: log: (pid 3505) 5177s
run: gitaly: (pid 4357) 5105s; run: log: (pid 2680) 5280s
run: gitlab-exporter: (pid 4366) 5105s; run: log: (pid 3264) 5194s
run: gitlab-workhorse: (pid 4322) 5106s; run: log: (pid 3113) 5214s
run: grafana: (pid 4579) 5094s; run: log: (pid 4120) 5123s
run: logrotate: (pid 4252) 1693s; run: log: (pid 2576) 5292s
run: nginx: (pid 3158) 5210s; run: log: (pid 3176) 5207s
run: node-exporter: (pid 4334) 5106s; run: log: (pid 3228) 5200s
run: postgres-exporter: (pid 4572) 5094s; run: log: (pid 3716) 5159s
run: postgresql: (pid 2827) 5274s; run: log: (pid 2848) 5271s
run: prometheus: (pid 4383) 5104s; run: log: (pid 3463) 5182s
run: puma: (pid 3025) 5228s; run: log: (pid 3036) 5225s
run: redis: (pid 2609) 5287s; run: log: (pid 2617) 5286s
run: redis-exporter: (pid 4368) 5105s; run: log: (pid 3359) 5188s
run: sidekiq: (pid 3053) 5222s; run: log: (pid 3067) 5219s

$ lsof -i:80
COMMAND  PID       USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   3158       root    7u  IPv4  28763      0t0  TCP *:http (LISTEN)
nginx   3159 gitlab-www    7u  IPv4  28763      0t0  TCP *:http (LISTEN)
nginx   3160 gitlab-www    7u  IPv4  28763      0t0  TCP *:http (LISTEN)
nginx   3161 gitlab-www    7u  IPv4  28763      0t0  TCP *:http (LISTEN)
nginx   3162 gitlab-www    7u  IPv4  28763      0t0  TCP *:http (LISTEN)
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220111183425464.png)