# GitLab Update(RPM)

> GitLab 跨版本升级，在开始升级之前，一定要做好备份工作，并记录好版本号。

1）查看 Gitlab 版本号，以现 11.1.4 版本为例；

```shell
$ cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
11.1.4
```
2）备份当前的 Gitlab

```shell
$ gitlab-rake gitlab:backup:create
```
**注意：除了在 `/var/opt/gitlab/backups` 下生成一个备份文件外，还需备份 `/etc/gitlab/gitlab-secrets.json`**

3）升级版本

**Gitlab 的升级不能跨越大版本号，需要升级到当前大版本号的最高版本，方可升级到下一个大版本号，如：11.1.4-->11.11.8-->12.7.5，先准备好相关的 rpm 包(在此选用清华源演示)**

```shell
$ wget https://mirror.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-11.11.8-ce.0.el7.x86_64.rpm
$ wget https://mirror.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-12.7.5-ce.0.el7.x86_64.rpm
$ wget https://mirror.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-12.9.2-ce.0.el7.x86_64.rpm
```
依次执行下面指令逐步升级，在每一步安装成功后如果发现界面 500，这是因为 redis 等程序还没完成启动，需要等会再访问即可，或清理 Redis 缓存：`gitlab-rake cache:clear`

（一定要保证数据可以正常访问方可执行下一步升级指令）

```shell
$ yum localinstall -y gitlab-ce-11.11.8-ce.0.el7.x86_64.rpm
$ yum localinstall -y gitlab-ce-12.7.5-ce.0.el7.x86_64.rpm
```

完成之后再查看当前版本：
```shell
$ cat /opt/gitlab/embedded/service/gitlab-rails/VERSION 
12.7.5
```

还原 `/etc/gitlab/gitlab-secrets.json`，如有配置请注意备份 gitlab.rb 文件。

```bash
$ cp -rf gitlab-secrets.json /etc/gitlab/gitlab-secrets.json
```

至此，Gitlab 的升级就完成了
