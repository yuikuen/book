# NextCloud Ex-Storage

> NextCloud 开启外部存储功能

NextCloud 默认是没有开启外部存储的功能，需要手动启用插件，并配置相关设置

1）点击头像并选中“应用”，启用 `External Storage Support`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903170338.png)

2）启用后，点击 **设置 – 左侧管理 – 外部存储** ，即可添加存储设备

> 虽然应用已添加成功，但还是提示 `“smbclient” 未安装。无法挂载 "SMB / CIFS", "SMB / CIFS 使用 OC 登录信息"。请联系管理员安装。`

- 方法一：编译安装

```bash
$ wget https://pecl.php.net/get/smbclient-1.0.6.tgz
$ tar -xf smbclient-1.0.6.tgz && cd ./smbclient-1.0.6
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config 
$ make && make install
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20190902/

$ vim /usr/local/php/etc/php.ini
extension=smbclient
```

- 方法二：通过 PECL 安装

```bash
$ yum install libsmbclient libsmbclient-devel -y
```

**安装完后，仅仅只是系统底层支持 smb client 了，而 nextcloud 是基于 php 的，还需要通过 PECL 命令来安装 php 对应的 smb 扩展**

```bash
$ pecl install smbclient
$ pecl channel-update pecl.php.net

$ vim /usr/local/php/etc/php.ini
extension=smbclient
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903170834.png)