# Nginx Issue

> 主要记录一些特殊的问题

## 问题一

`错误提示：nginx[4115]: nginx: [emerg] open() "/var/run/nginx/nginx.pid" failed (2: No such file or directory) `

**场景** 每次服务器重启后，`/var/run/nginx` 目录都会被删除，所以无法在这目录创建 nginx.pid文件，可以再次创建此目录，然后就可以运行了，但重启后又会丢失

**解决方法**：打开配置文件，打开其中一个配置，在 nginx 安装目录下创建 logs 文件

```bash
$ vim /usr/local/nginx/conf/nginx.conf
# 取消注释，打开此配置
pid        logs/nginx.pid;

$ mkdir -p /usr/local/nginx/logs
```

具体原因：在配置文件显示的指定 nginx.pid 文件存放位置，然后创建 logs 文件夹，当服务器重启后，logs 目录不会被删除。


