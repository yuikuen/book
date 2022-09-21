# GitLab Backup

> GitLab 备份还原过程

## 手动备份

gitlab 默认备份路径为 `/var/opt/gitlab/backups`，而修改 `gitlab.rb` 配置文件，可自定义本地备份存放路径(非必须)

```sh
### Backup Settings
###! Docs: https://docs.gitlab.com/omnibus/settings/backups.html
# 常用备份配置如下
gitlab_rails['manage_backup_path'] = true
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"  # gitlab 备份目录
gitlab_rails['backup_archive_permissions'] = 0644        # 生成的备份文件权限
gitlab_rails['backup_keep_time'] = 604800                # 备份保留天数7天（604800秒）
```

执行如下命令生成 gitlab 备份

```sh
$ gitlab-rake gitlab:backup:create
-rw------- 1 chrony polkitd 1327226880 Aug  3 01:24 1627925062_2021_08_03_13.3.5_gitlab_backup.tar

$ cp /etc/gitlab/gitlab-secrets.json gitlab-secrets.json.$(date +%F_%T)
```

## 定时备份

结合 crontab 实施自动定时备份，比如每天0点、6点、12点、18点各备份一次

```sh
$ cd /var/opt/gitlab/backups
$ vim gitlab_backup.sh
#!/bin/bash
/usr/bin/gitlab-rake gitlab:backup:create CRON=1
# 环境变量 CRON=1 的作用是如果没有任何错误发生时，抑制备份脚本的所有进度输出
$ crontab -e
0 0,6,12,18 * * * /bin/bash -x /var/opt/gitlab/backups/gitlab_backup.sh > /dev/null 2>&1
```

```bash
#!/bin/bash
#定时任务在crontab

dateTime=`date +%Y-%m-%d`                  #当前系统时间
bakdata=$dateTime.tar.gz                   #打包文件名
bakname=/home/gitlab/data/backups/*.tar    #备份的内容
SOURCE_PATH=/home/gitlab/data/backups      #本地备份目录
DEST_PATH=/home/backup/ZS-Backup/gitlab    #远程服务器备份目录

echo "----------进入备份目录..."
cd $SOURCE_PATH

echo "----------删除本机过期数据..."
find $SOURCE_PATH -name "*.tar" -ctime +1 -type f -exec rm -rf {} \;
find $SOURCE_PATH -name "*.tar.gz" -ctime +1 -type f -exec rm -rf {} \;

echo "----------备份Gitlab数据..."
docker exec -t gitlab gitlab-rake gitlab:backup:create CRON=1;
tar -zcvf $bakdata $bakname

echo "----------备份数据至远程目录"
cp -R $SOURCE_PATH/*.tar.gz $DEST_PATH

echo "----------删除远程目录过期数据..."
find $DEST_PATH -name "*.tar.gz" -ctime +7 -type f -exec rm -rf {} \;

echo -e "-------备份结束\n"
exit
```

## 备份还原

通过备份，可实现代码数据迁移和数据还原等操作，将 backups 中最近的一次备份，复制到另一台 Gitlab 服务器

```bash
# 复制到同样的路径
$ cd /var/opt/gitlab/backups

# 给予备份权限，并停止连接数据库的进程，unicorn是rubby相关的webserver，sidekiq是rubby相关的消息队列
$ chmod 777 1580716769_2020_02_03_12.7.5_gitlab_backup.tar
$ gitlab-ctl stop unicorn
$ gitlab-ctl stop sidekiq

# 使用命令还原数据，中途按提示输入Yes即可
$ gitlab-rake gitlab:backup:restore BACKUP=1627925062_2021_08_03_13.3.5
$ cp -rf gitlab-secrets.json /etc/gitlab/gitlab-secrets.json 
$ gitlab-ctl reconfigure
$ gitlab-ctl restart
$ gitlab-ctl status
```

在使用 `gitlab-ctl reconfigure` 前，需要复制备份好的 `gitlab-secrets.json` 文件到`/etc/gitlab/`目录进行覆盖，因为 Gitlab 密钥文件-此密钥包含了数据库加密密钥和密钥变量。

如不能恢复此文件，那么用户的密码就无法访问 gitlab 服务器，或者打开 gitlab 项目会报 500 错误。

**注：如需作服务器迁移还原，注意将 `gitlab.rb` 文件同步复制到新服务器，因为内有 gitlab 域名信息，也就是服务器地址，版本一致的情况下，可覆盖并还原原服务器配置；**

另需要注意的是：新服务器上的 Gitlab 的版本必须与创建备份时的 Gitlab 版本号相同，比如新服务器安装的是最新的 14 版本的 Gitlab, 那么迁移之前, 最好将老服务器的 Gitlab 升级为 14 再进行备份