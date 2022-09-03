# NextCloud Cron

> NextCloud 配置后台任务-官方推荐后台任务采用 Cron 方式

使用操作系统 `Cron` 功能是执行常规任务的首选方法，此方法可以执行预定作业，而不会受到 Web 服务器可能具有的固有限制。

但在 CentOS7 中并未起作用，可能因为需要调用 php-cli 而不是 php，所以使用 systemd 定时器替代 cron 作业，具体可参考 [官方文档](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/background_jobs_configuration.html?highlight=cron)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903165101.png)

1）在 `/etc/systemd/system/` 创建这两个文件

```bash
$ vim /etc/systemd/system/nextcloudcron.service
[Unit]
Description=Nextcloud cron.php job

[Service]
User=nginx
ExecStart=/usr/local/php/bin/php -f /usr/local/nginx/html/cron.php
KillMode=process
```

```bash
$ vim /etc/systemd/system/nextcloudcron.timer
[Unit]
Description=Run Nextcloud cron.php every 5 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Unit=nextcloudcron.service

[Install]
WantedBy=timers.target
```

```bash
$ systemctl enable --now nextcloudcron.timer
```

通过配置 `nextcloud cron .timer` 每 5 分钟执行一次 `php -f cron.php` 任务