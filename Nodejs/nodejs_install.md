# Nodejs Install

> CentOS7.x 安装 Nodejs

1）到 [Nodejs 中文网](http://nodejs.cn/download/) 查看下载地址，并且下载相应的安装包；

```bash
$ wget https://npm.taobao.org/mirrors/node/v16.9.1/node-v16.9.1-linux-x64.tar.xz

$ xz -d node-v16.9.1-linux-x64.tar.xz
$ tar -xf node-v16.9.1-linux-x64.tar -C /usr/local/nodejs
```

2）建立软连接，变为全局

```bash
$ ln -s /usr/local/nodejs/node-v16.9.1-linux-x64/bin/npm /usr/local/bin/
$ ln -s /usr/local/nodejs/node-v16.9.1-linux-x64/bin/node /usr/local/bin/
```

```bash
$ vim /etc/profile
# 在文末增加下列内容
# nodejs
export NODE_HOME=/usr/local/nodejs/node-v16.9.1-linux-x64
export PATH=$NODE_HOME/bin:$PATH
```

3）配置生效，并测试

```bash
$ source /etc/profile
$ node -v
v16.9.1
$ npm -v
6.14.15
```
