# RabbitMQ Install(Source)

> 源码安装 RabbitMQ 3.9.x

## 版本环境

RabbitMQ 采用 Erlang 语言开发，安装前建议先查看 [Erlang官方说明](https://www.rabbitmq.com/which-erlang.html) 并注意 Readme 上的版本兼容问题

- System：CentOS7.9.2009 Minimal
- Erlang：Erlang-24.2
- RabbitMQ：RabbitMQ_Server-3.9.14

## 安装 Erlang

> 源码编译安装，[Erlang 官方下载地址](tp://www.erlang.org/download)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220419155223.png)

1）下载相关依赖及组件

```bash
$ wget https://erlang.org/download/otp_src_24.2.tar.gz
$ yum -y install make gcc gcc-c++ kernel-devel m4 ncurses-devel openssl-devel gtk2-devel binutils-devel unixODBC unixODBC-devel xz socat
```

2）解压编译安装

```bash
$ tar -zxvf otp_src_24.2.tar.gz && cd otp_src_24.2
$ ./configure prefix=/opt/erlang
$ make && make install
```

3）配置环境变量并验证

```bash
$ echo 'export PATH=/opt/erlang/bin:$PATH' >> /etc/profile
$ source /etc/profile
$ erl -version
Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 12.2
```

## 安装 RabbitMQ

> 源码编译安装，[RabbitMQ 官方下载地址](tps://www.rabbitmq.com/download.html)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220419155441.png)

1）下载组件并配置环境变量

```bash
$ wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.9.14/rabbitmq-server-generic-unix-3.9.14.tar.xz
$ tar -xvf rabbitmq-server-generic-unix-3.9.14.tar.xz -C /opt/
$ cd /opt && mv rabbitmq_server-3.9.14 rabbitmq

$ echo 'export PATH=/opt/rabbitmq/sbin:$PATH' >> /etc/profile
$ source /etc/profile
```

2）创建账户并赋权

```bash
$ hostnamectl set-hostname rabbitmq && bash
$ echo '127.0.0.1 rabbitmq' >> /etc/hosts

$ useradd rabbitmq
$ mkdir -p /opt/rabbitmq/var/lib/rabbitmq /opt/rabbitmq/var/log/rabbitmq
$ chown rabbitmq.rabbitmq -R /opt/rabbitmq
```

3）创建配置文件

```bash
# 添加浏览器管理插件，默认安装是没有管理页面
$ cat >> /opt/rabbitmq/etc/rabbitmq/enabled_plugins <<EOF
[rabbitmq_management].
EOF

$ cat >> /opt/rabbitmq/etc/rabbitmq/rabbitmq-env.conf <<EOF
RABBITMQ_NODENAME=rabbit@rabbitmq
RABBITMQ_NODE_IP_ADDRESS=0.0.0.0
RABBITMQ_NODE_PORT=5672
RABBITMQ_LOG_BASE=/opt/rabbitmq/var/log/rabbitmq
RABBITMQ_MNESIA_BASE=/opt/rabbitmq/var/lib/rabbitmq/mnesia
EOF

$ cat >> /opt/rabbitmq/etc/rabbitmq/rabbitmq.conf <<EOF
listeners.tcp.default = 5672
num_acceptors.tcp = 10

management.tcp.port = 15672
management.tcp.ip   = 0.0.0.0
management.http_log_dir = /opt/rabbitmq/var/log/rabbitmq/management_access
	
vm_memory_high_watermark.absolute = 512MiB
vm_memory_high_watermark_paging_ratio = 0.3

loopback_users.guest = true
EOF
```

4）创建开机自启服务

```bash
$ cat >> /etc/systemd/system/rabbitmq-server.service <<EOF
[Unit]
Description=RabbitMQ broker
After=syslog.target network.target

[Service]
Type=notify
User=rabbitmq
Group=rabbitmq
UMask=0027
NotifyAccess=all
TimeoutStartSec=3600
LimitNOFILE=32768
Restart=on-failure
RestartSec=10
WorkingDirectory=/opt/rabbitmq/var/lib/rabbitmq
ExecStart=/opt/rabbitmq/sbin/rabbitmq-server
ExecStop=/opt/rabbitmq/sbin/rabbitmqctl shutdown
SuccessExitStatus=69

[Install]
WantedBy=multi-user.target
EOF
```

5）设置 Cookie 并启动服务

```bash
$ echo "rabbitmq-cluster-cookie" >> ~/.erlang.cookie
$ echo "rabbitmq-cluster-cookie" >> /home/rabbitmq/.erlang.cookie
$ chown rabbitmq.rabbitmq /home/rabbitmq/.erlang.cookie
$ chmod 600 ~/.erlang.cookie /home/rabbitmq/.erlang.cookie
$ systemctl daemon-reload && systemctl enable --now rabbitmq-server
$ systemctl status rabbitmq-server
```

**注：如启动报错，请根据错误提示进行排查，如下问题：erl: command not found 环境没找到**

```bash
$ journalctl -xe
-- Unit rabbitmq-server.service has begun starting up.
Apr 22 14:30:47 rabbitmq rabbitmq-server[1507]: /opt/rabbitmq/sbin/rabbitmq-server: line 187: erl: command not found
Apr 22 14:30:47 rabbitmq systemd[1]: rabbitmq-server.service: main process exited, code=exited, status=127/n/a
Apr 22 14:30:47 rabbitmq systemd[1]: Failed to start RabbitMQ broker.
-- Subject: Unit rabbitmq-server.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- 
-- Unit rabbitmq-server.service has failed.
```

解决方法：增加 erl 环境变量指定 `export PATH=/opt/erlang/bin:$PATH`

```bash
$ vim /opt/rabbitmq/sbin/rabbitmq-server +187
185 # NOTIFY_SOCKET is needed here to prevent epmd from impersonating the
186 # success of our startup sequence to systemd.
# 添加erlang的目录路径
187 export PATH=/opt/erlang/bin:$PATH
188 NOTIFY_SOCKET= \
189 RABBITMQ_CONFIG_FILE=$RABBITMQ_CONFIG_FILE \
190 ERL_CRASH_DUMP=$ERL_CRASH_DUMP \
191 RABBITMQ_CONFIG_ARG_FILE=$RABBITMQ_CONFIG_ARG_FILE \
192 RABBITMQ_DIST_PORT=$RABBITMQ_DIST_PORT \
193     ${ERL_DIR}erl -pa "$RABBITMQ_EBIN_ROOT" \
194     -boot "${CLEAN_BOOT_FILE}" \
195     -noinput \
196     -hidden \
197     -s rabbit_prelaunch \
198     ${RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS} \
199     ${RABBITMQ_NAME_TYPE} ${RABBITMQ_PRELAUNCH_NODENAME} \
200     -conf_advanced "${RABBITMQ_ADVANCED_CONFIG_FILE}" \
201     -rabbit feature_flags_file "\"$RABBITMQ_FEATURE_FLAGS_FILE\"" \
202     -rabbit enabled_plugins_file "\"$RABBITMQ_ENABLED_PLUGINS_FILE\"" \
203     -rabbit plugins_dir "\"$RABBITMQ_PLUGINS_DIR\"" \
204     -extra "${RABBITMQ_NODENAME}"
```

6）添加管理账户并分配权限

```bash
# 检查用户列表
$ rabbitmqctl list_users
Listing users ...
user	tags
guest	[administrator]
rabbitmq_user	[]

# RabbitMQ 有默认用户密码，guest/guest该用户密码只能在本地登陆，若在浏览器中登陆，须创建新用户密码
$ rabbitmqctl add_user rabbitmq_user rabbitmq_pwd
Adding user "rabbitmq_user" ...
Done. Don't forget to grant the user permissions to some virtual hosts! See 'rabbitmqctl help set_permissions' to learn more.

# 为rabbitmq_user用户添加管理员角色
$ rabbitmqctl set_user_tags rabbitmq_user administrator 
Setting tags for user "rabbitmq_user" to [administrator] ...

# 设置rabbitmq_user用户权限，允许访问vhost及read/write
$ rabbitmqctl set_permissions -p / rabbitmq_user ".*" ".*" ".*"
Setting permissions for user "rabbitmq_user" in vhost "/" ...

# 检查权限列表
$ rabbitmqctl list_permissions -p /
Listing permissions for vhost "/" ...
user	configure	write	read
rabbitmq_user	.*	.*	.*
guest	.*	.*	.*
```

7）浏览器登录访问 http://rabbitmq-server:15672 

注：若出现 `ReferenceError:disable_stats is not defined`，可 `ctrl+f5` 清除页面缓存后重新登陆

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220419170741.png)