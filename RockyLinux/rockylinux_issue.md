# RockyLinux Issue

> 问题记录

## Root-SSH 登录失败

Rocky Linux 9 版本中，为了增加安全性，默认是禁用 SSH root 密码登录，需要自行修改配置文件。

```bash
$ vim /etc/ssh/sshd_config
# 取消注释并修改
#PermitRootLogin prohibit-password
PermitRootLogin yes

$ systemctl restart sshd
```