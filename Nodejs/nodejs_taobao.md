# Nodejs Taobao

> 在使用 npm 下载一些插件包的时候，会有很多插件下载不下来，这时需要配置一下淘宝镜像源地址，这样就可以从国内淘宝的镜像地址下载

1）清理缓存

```bash
$ npm cache clean --force
```

2）切换为淘宝镜像，注意**原淘宝 npm 域名即将停止解析**

[镜像站说明](https://developer.aliyun.com/mirror/NPM?spm=a2c6h.25603864.0.0.6af14ccaxmcKBh) 表明`http://npm.taobao.org`和 `http://registry.npm.taobao.org` 将在 **2022.06.30** 号正式下线和停止 DNS 解析

```bash
$ npm config set registry http://registry.npmmirror.com
$ npm config get registry http://registry.npmmirror.com
```

3）输入命令查看是否成功切换

```bash
$ npm config list
```

4）最后：直接配置全局使用 cnpm 命令下载所有插件包

```bash
$ npm install --global cnpm
$ cnpm install
```

以后要下载插件包把 npm 改成 cnpm 便行