# Gitlab(rpm)安装14.1.0

## 前置环境

1)开放对应服务协议，或直接禁用防火墙

```bash
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service

$ firewall-cmd --add-service=ssh --permanent
$ firewall-cmd --add-service=http --permanent
$ firewalld-cmd --reload
```

2)配置依赖环境

```bash
$ yum install policycoreutils openssh-server openssh-clients postfix
$ systemctl enable --now sshd postfix
```

## 目录说明

GitLab由主要由以下服务构成，他们共同承担了Gitlab的运作需要：
- nginx: 静态web服务器
- gitlab-shell: 用于处理Git命令和修改authorized keys列表
- gitlab-workhorse: 轻量级的反向代理服务器
- logrotate：日志文件管理工具
- postgresql：数据库
- redis：缓存数据库
- sidekiq：用于在后台执行队列任务（异步执行）
- unicorn：HTTP服务，GitLab Rails应用是托管在这个服务器上面的

主要程序目录：
- 主配置文件: /etc/gitlab/gitlab.rb
- 文档根目录: /opt/gitlab
- 默认存储库位置: /var/opt/gitlab/git-data/repositories
- Nginx配置文件: /var/opt/gitlab/nginx/conf/gitlab-http.conf
- Postgresql数据目录: /var/opt/gitlab/postgresql/data

## 安装程序

[清华源-GitLab]:https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/

![image-20220111165225877](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220111165225877.png)

- 方法一：配置源地址，再通过 yum 在线安装指定版本

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

- 方法二：下载 rpm 文件，再离线安装指定版本

```bash
$ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-14.1.0-ce.0.el7.x86_64.rpm
$ rpm -ivh gitlab-ce-14.1.0-ce.0.el7.x86_64.rpm --force --nodeps
```

## 配置程序

1)修改 `/etc/gitlab/gitlab.rb` 配置文件

```bash
## GitLab URL 可为域名或本机IP
external_url 'http://188.188.4.141'
```

2)初始化服务，每次修改配置后需要执行初始化命令并重启方可生效

```bash
$ gitlab-ctl reconfigure
$ gitlab-ctl restart
```

3)部署成功后，初始管理员用户为 root，密码在安装过程中随机生成并保存在 `/etc/gitlab/initial_root_password`，有效期为 24 小时

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

4)查看服务状态，并浏览器登录验证

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

![image-20220111183425464](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220111183425464.png)

## 基本操作

- 控制台打开：gitlab-rails
- 数据库命令：gitlab-psql
- 数据备份还原：gitlab-rake
- 客户端操作命令：gitlab-ctl
  - 启动：gitlab-ctl start
  - 停止：gitlab-ctl stop
  - 重启：gitlab-ctl restart
  - 查看：gitlab-ctl status
  - 日志：gitlab-ctl tail nginx(组件)

**参考链接：**
- [Install self-managed GitLab](https://about.gitlab.com/install/#already-installed)