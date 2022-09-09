# NextCloud Issue

> 记录一些运作中发现的问题
> 
> PS：修改各配置文件后，记得重启相关的服务方可生效

## 问题一

**一些文件未通过完整性检查**，通过无效信息可以得出 .htaccess，.user.ini 这二个文件是无效的

解决方法：删除 .htaccess，.user.ini 两个文件，然后从 nextcloud 下载文件，复制 .htaccess，.user.ini 到网站根目录

```bash
Technical information
=====================
The following list covers which files have failed the integrity check. Please read
the previous linked documentation to learn more about the errors and how to fix
them.

Results
=======
- core
	- INVALID_HASH
		- .htaccess
	- FILE_MISSING
		- .user.ini

Raw output
...

$ cp .htaccess .user.ini /usr/local/nginx/html/
```

## 问题二

**PHP 内存限制低于建议值 512MB**

```bash
$ vim /usr/local/php/etc/php.ini
# 找到 memory_limit = 128M，将128M修改为512M，数值按内存情况及需要而定，保存退出
; Maximum amount of memory a script may consume (128MB)
; http://php.net/memory-limit
memory_limit = 512M
```

## 问题三

**PHP configuration option output_buffering must be disabled**

```bash
$ vim /usr/local/php/etc/php.ini
# 找到 output_buffering 并;注释掉参数即可
; output_buffering = 4096
```

## 问题四

**PHP 的安装似乎不正确，无法访问系统环境变量。getenv("PATH") 函数测试返回了一个空值**

```bash
# 确认环境变量是否有配置成功
$ php -v
PHP 7.4.27 (cli) (built: Feb 14 2022 15:53:46) ( NTS )
Copyright (c) The PHP Group
Zend Engine v3.4.0, Copyright (c) Zend Technologies
```

- 方法一：

```bash
$ vim /usr/local/php/etc/php-fpm.conf
# 在文中最后添加
env[PATH] = /usr/local/bin:/usr/bin:/bin:/usr/local/php/bin
```

- 方法二：

```bash
$ vim /usr/local/php/etc/php-fpm.d/www.conf
# 去掉下面几行注释
env[HOSTNAME] = $HOSTNAME                     
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
```

## 问题五

**"Strict-Transport-Security" HTTP 头未设为至少 "15552000" 秒。为了提高安全性，建议启用 HSTS**

```bash
$ vim /usr/local/nginx/vhosts/nextcloud.conf
    # HSTS settings
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    # 取消下参数注释即可
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
```

## 问题六

**您的安装没有设置默认的电话区域**，编辑 Nextcloud config 目录中的 config.php 文件，在文件最下方，`);` 前添加如下代码

```bash
$ vim /usr/local/nginx/html/config/config.php
  'default_phone_region' => 'CN',
);
```

## 问题七

**内存缓存未配置。为了提升性能，请尽量配置内存缓存**，此问题也会产生"Internal Server Error" 错误，原因在于设置了 Redis ，但未安装 php-redis 扩展

```bash
$ wget https://github.com/phpredis/phpredis/archive/refs/tags/5.3.7RC2.tar.gz -O phpredis-5.3.7RC2.tar.gz
$ tar -xf phpredis-5.3.7RC2.tar.gz && cd ./phpredis-5.3.7RC2
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config
$ make && make install
$ vim /usr/local/php/etc/php.ini
extension=redis
```
```bash
$ vim /usr/local/nginx/html/config/config.php
<?php
$CONFIG = array (
  'instanceid' => 'ocpyykjf3xcl',
  'passwordsalt' => 'heutk+7YcRIJizzXL3UCp8IH7y0YQb',
  'secret' => 'wtzj8FhDy2HjuoAhWQ1J3OFwpior+gC1dVQj/xQMr8YBgRWf',
  'trusted_domains' =>
  array (
    0 => '188.188.4.254',
  ),
  # 增加下述Redis内容，！！注意此注释不可出现在配置中，记得删除，否则会报错！！
  'memcache.local' => '\OC\Memcache\Redis',
  'memcache.distributed' => '\OC\Memcache\Redis',
  'memcache.locking' => '\OC\Memcache\Redis',
  'redis' => array(
    'host' => 'localhost',
    'port' => 6379,
  ),
  'datadirectory' => '/usr/local/nginx/html/data',
  'dbtype' => 'mysql',
  'version' => '24.0.4.1',
  'overwrite.cli.url' => 'http://188.188.4.254',
  'dbname' => 'cloud_db',
  'dbhost' => '127.0.0.1:3306',
  'dbport' => '',
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'oc_YuiKuen.Yuen',
  'dbpassword' => 'ZxdkbqEoN1U5jVyHRqUSWdF6c9V5uY',
  'installed' => true,
  'default_phone_region' => 'CN',
  'mail_smtpmode' => 'smtp',
  'mail_smtpsecure' => 'ssl',
  'mail_sendmailmode' => 'smtp',
  'mail_from_address' => '298087440',
  'mail_domain' => 'qq.com',
  'mail_smtpauthtype' => 'LOGIN',
  'mail_smtphost' => 'smtp.qq.com',
  'mail_smtpport' => '465',
  'mail_smtpauth' => 1,
  'mail_smtpname' => '298087440@qq.com',
  'mail_smtppassword' => 'ifgmrwlnriwjcacf',
);
```

## 问题八

**The PHP OPcache module is not loaded (PHP 的 OPcache 模块未载入)**

```bash
$ cd /usr/local/src/php-7.4.27/ext/opcache
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config 
$ make && make install
# make install生成文件路径地址，抄写绝对路径；
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20190902/

# 如果zend_extension = opcache.so不生效，请填写绝对路径
$ vim /usr/local/php/etc/php.ini
zend_extension = opcache.so
opcache.enable=1
opcache.enable_cli=1
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1
```

## 问题九

**该实例缺失了一些推荐的 PHP 模块**

- intl

```bash
$ cd /usr/local/src/php-7.4.27/ext/intl
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config 
$ make && make install
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20190902/

$ vim /usr/local/php/etc/php.ini
extension=intl
```

- bcmath

```bash
$ cd /usr/local/src/php-7.4.27/ext/bcmath
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config 
$ make && make install
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20190902/

$ vim /usr/local/php/etc/php.ini
extension=bcmath
```

- gmp

```bash
$ cd /usr/local/src/php-7.4.27/ext/gmp
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config 
$ make && make install
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20190902/
Installing header files:          /usr/local/php/include/php/

$ vim /usr/local/php/etc/php.ini
extension=gmp
```

- imagick

```bash
# 方法一：先安装 imageMagic，再安装 imagick 扩展
$ yum install -y ImageMagick*
$ wget https://pecl.php.net/get/imagick-3.7.0.tgz
$ tar -xf imagick-3.7.0.tgz && cd ./imagick-3.7.0
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config
$ make && make install
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20190902/
Installing header files:          /usr/local/php/include/php/

$ vim /usr/local/php/etc/php.ini
extension=imagick

# 方法二：编译安装，便于后期维护
$ wget https://download.imagemagick.org/ImageMagick/download/ImageMagick-7.1.0-24.tar.gz
$ tar -xf ImageMagick-7.1.0-24.tar.gz && cd ./ImageMagick-7.1.0-24
$ ./configure --prefix=/usr/local/imagemagick
$ make && make install
$ /usr/local/imagemagick/bin/convert -version
Version: ImageMagick 7.1.0-24 Q16-HDRI x86_64 2022-02-12 https://imagemagick.org
Copyright: (C) 1999-2021 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC HDRI OpenMP(4.5) 
Delegates (built-in): bzlib freetype jng jpeg lzma png xml zip zlib
Compiler: gcc (9.3)

$ wget https://pecl.php.net/get/imagick-3.7.0.tgz
$ tar -xf imagick-3.7.0.tgz && cd ./imagick-3.7.0
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config
$ make && make install
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20190902/
Installing header files:          /usr/local/php/include/php/

$ vim /usr/local/php/etc/php.ini
extension=imagick
```

- sodium

```bash
configure: error: Package requirements (libsodium >= 1.0.8) were not met:
No package 'libsodium' found
$ wget https://download.libsodium.org/libsodium/releases/libsodium-1.0.18-stable.tar.gz
$ tar -xf libsodium-1.0.18-stable.tar.gz && cd ./libsodium-stable
$ ./configure CC="gcc -m64" --prefix=/usr --libdir=/usr/lib64
$ make && make install
$ ldconfig

$ cd /usr/local/src/php-7.4.27/ext/sodium
$ /usr/local/php/bin/phpize
$ ./configure --with-php-config=/usr/local/php/bin/php-config 
$ make && make install
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20190902/

$ vim /usr/local/php/etc/php.ini
extension=sodium
```

所有问题修复完后，刷新即提示"所有检查已通过"

## 问题十

**通过不被信任的域名访问**

NextCloud 在访问时，会自动判断已设置好的域名或 IP 是否被允许，如是固定 IP 的话，直接把域名或 IP 添加到配置文件即可。如是家用自建服务器，路由会每天都重置一次 IP，导致上述问题，通过下述方法可禁止 IP 限制

```bash
$ vim /usr/local/nginx/html/config/config.php
<?php
$CONFIG = array (
  'instanceid' => 'ocpyykjf3xcl',
  'passwordsalt' => 'heutk+7YcRIJizzXL3UCp8IH7y0YQb',
  'secret' => 'wtzj8FhDy2HjuoAhWQ1J3OFwpior+gC1dVQj/xQMr8YBgRWf',
  'trusted_domains' =>
  array (
    0 => '188.188.4.254',
    # 增加公网IP或域名
    1 => '183.x.x.x',
    2 => 'cloud.yuikuen.top',
    # 将当前访问的域名或IP动态的添加的信任的域名中
    3 => preg_match('/cli/i',php_sapi_name())?'127.0.0.1':$_SERVER['SERVER_NAME'],
  ),
```

## 问题十一

**预览状态下无法生成缩略图**

Nextcloud 支持图片文件、MP3 文件的封面和文本文件的预览。但默认并不会为存储的图片提前生成缩略图，只有当在网页或客户端，访问到相应的图片时，才会在服务器上进行生成。此生成的策略，一定程度上节省了服务器的空间，但在管理上或同时访问下会加载特别慢。

1）首先设置文件类型的支持和预览参数

- 方法一：增加预览文件的支持及相关参数（推荐）

```bash
$ vim /usr/local/nginx/html/config/config.php
  'enable_previews' => true,
  //最大预览尺寸
  'preview_max_x' => 2000,  //预览的最大宽度，以像素为单位。值null表示没有限制
  'preview_max_y' => 2000,  //预览的最大高度，以像素为单位。值null表示没有限制
  'preview_max_filesize_image' => 50,  //生成图像预览的最大文件大小,默认50M
  'preview_max_memory' => 128,         //创建图像会分配的内存默认128M
  //图片大小缩放
  'preview_max_scale_factor' => 10,    //禁用缩放改成1，禁用最大比例因子改为null
  'enabledPreviewProviders' =>         //开启支持的格式
  array (
    0 => 'OC\\Preview\\PNG',
    1 => 'OC\\Preview\\JPEG',
    2 => 'OC\\Preview\\GIF',
    3 => 'OC\\Preview\\BMP',
    4 => 'OC\\Preview\\XBitmap',
    5 => 'OC\\Preview\\MP3',
    6 => 'OC\\Preview\\MarkDown',
    7 => 'OC\\Preview\\OpenDocument',
    8 => 'OC\\Preview\\Krita',
    9 => 'OC\\Preview\\HEIC',
   10 => 'OC\\Preview\\Movie',
   11 => 'OC\\Preview\\MP4',
   12 => 'OC\\Preview\\AVI',
   13 => 'OC\\Preview\\MKV',
  ),
```

- 方法二：**手动设置缩略图参数**，与上述内容一致

```bash
# 预览图像的默认 JPEG 质量设置为“90”
$ cd /usr/local/nginx/html
$ occ config:app:set preview jpeg_quality --value="60"
//参考内容
$ sudo -u apache php occ config:app:set previewgenerator squareSizes --value="64 256 1024"
$ sudo -u apache php occ config:app:set previewgenerator widthSizes  --value="64 256"
$ sudo -u apache php occ config:app:set previewgenerator heightSizes --value="64 256"
$ sudo -u apache php occ config:system:set preview_max_x --value 2000
$ sudo -u apache php occ config:system:set preview_max_y --value 2000
$ sudo -u apache php occ config:app:set preview jpeg_quality --value="60"
```

2）安装 `Preview Generator` 插件并实现自动生成图片缩略图

> 注意：此方法是以空间换时间的方式，提前生成好了缩略图，节省了需要用的时候的 cpu 计算时间，但是也占用了磁盘空间 (需要提前做好上述预览配置，减少文件像素大小，不然占用空间较大，占用的空间都好几十个G上涨)

按照提示执行命令，因为 nextcloud 之前安装的权限为 nginx，因此由用户身份 nginx 执行

- 手动触发任务

```bash
$ sudo -u nginx php occ preview:generate-all -vvv
$ sudo -u nginx php /usr/local/nginx/html/occ files:scan --all
```
缩略图会被存放在  `/usr/local/nginx/html/data/appdata_oc87xfzn8xfn/preview` 目录下

- 设置定时任务自动触发任务

```bash
# 当有新图片上传时，插件并不会自动触发，需要自行调用，可以将此命令加入到定时任务中
$ crontab -e
*/15 * * * * sudo -u nginx php /usr/local/nginx/html/occ preview:generate -vvv
*/16 * * * * sudo -u nginx php /usr/local/nginx/html/occ files:scan --all
```

3）查看缩略图占用情况

```bash
$ /usr/local/nginx/html/data/appdata_oc87xfzn8xfn/preview
$ du -h --max-depth=1 .
1.7M	./e
2.0M	./c
1.9M	./1
708K	./7
1.1M	./9
1.5M	./6
1.5M	./b
764K	./8
2.0M	./3
1.3M	./a
828K	./2
532K	./0
256K	./5
760K	./d
956K	./f
488K	./4
18M	.
```

4）设置视频生成预览图

> 视频文件需要另外安装程序 `FFmpeg` 方可实现

```bash
$ wget https://johnvansickle.com/ffmpeg/builds/ffmpeg-git-amd64-static.tar.xz
$ tar -xf ffmpeg-git-amd64-static.tar.xz && cd ./ffmpeg-git-20220108-amd64-static
$ mv ffmpeg ffprobe /usr/bin/

ffmpeg -version
ffmpeg version N-60236-gffb000fff8-static https://johnvansickle.com/ffmpeg/  Copyright (c) 2000-2022 the FFmpeg developers
built with gcc 8 (Debian 8.3.0-6)
configuration: --enable-gpl --enable-version3 --enable-static --disable-debug --disable-ffplay --disable-indev=sndio --disable-outdev=sndio --cc=gcc --enable-fontconfig --enable-frei0r --enable-gnutls --enable-gmp --enable-libgme --enable-gray --enable-libaom --enable-libfribidi --enable-libass --enable-libvmaf --enable-libfreetype --enable-libmp3lame --enable-libopencore-amrnb --enable-libopencore-amrwb --enable-libopenjpeg --enable-librubberband --enable-libsoxr --enable-libspeex --enable-libsrt --enable-libvorbis --enable-libopus --enable-libtheora --enable-libvidstab --enable-libvo-amrwbenc --enable-libvpx --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxml2 --enable-libdav1d --enable-libxvid --enable-libzvbi --enable-libzimg
libavutil      57. 18.100 / 57. 18.100
libavcodec     59. 20.100 / 59. 20.100
libavformat    59. 17.100 / 59. 17.100
libavdevice    59.  5.100 / 59.  5.100
libavfilter     8. 25.100 /  8. 25.100
libswscale      6.  5.100 /  6.  5.100
libswresample   4.  4.100 /  4.  4.100
libpostproc    56.  4.100 / 56.  4.100

$ ffprobe -version
ffprobe version N-60236-gffb000fff8-static https://johnvansickle.com/ffmpeg/  Copyright (c) 2007-2022 the FFmpeg developers
built with gcc 8 (Debian 8.3.0-6)
configuration: --enable-gpl --enable-version3 --enable-static --disable-debug --disable-ffplay --disable-indev=sndio --disable-outdev=sndio --cc=gcc --enable-fontconfig --enable-frei0r --enable-gnutls --enable-gmp --enable-libgme --enable-gray --enable-libaom --enable-libfribidi --enable-libass --enable-libvmaf --enable-libfreetype --enable-libmp3lame --enable-libopencore-amrnb --enable-libopencore-amrwb --enable-libopenjpeg --enable-librubberband --enable-libsoxr --enable-libspeex --enable-libsrt --enable-libvorbis --enable-libopus --enable-libtheora --enable-libvidstab --enable-libvo-amrwbenc --enable-libvpx --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxml2 --enable-libdav1d --enable-libxvid --enable-libzvbi --enable-libzimg
libavutil      57. 18.100 / 57. 18.100
libavcodec     59. 20.100 / 59. 20.100
libavformat    59. 17.100 / 59. 17.100
libavdevice    59.  5.100 / 59.  5.100
libavfilter     8. 25.100 /  8. 25.100
libswscale      6.  5.100 /  6.  5.100
libswresample   4.  4.100 /  4.  4.100
libpostproc    56.  4.100 / 56.  4.100
```

![image-20220216143729570](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220216143729570.png)

## 问题十二

> 注意：升级之前，首先要考虑备份数据库

**NextCloud 提醒信息一直提示版本可升级，通过其后台进行自动更新，但当前服务器在国内，由于网络原因无法完成自动更新**

![image-20220518153404331](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220518153404331.png)

1）下载安装包并上传指定目录，路径一般为 `${NEXTCLOUD_PATH}/data/updater-XXXXXX/downloads`

```bash
$ wget https://download.nextcloud.com/server/releases/nextcloud-23.0.4.zip
$ mv nextcloud-23.0.4.zip ${NEXTCLOUD_PATH}/data/updater-ockwh32we132/downloads
```

2）修改状态设置，updater-xxxxxxxx 目录有一个隐藏文件 `.step` 文件，将里面的内容由`{"state":"start","step":4}`修改为`{"state":"stop","step":5}`，简单来说是要路过上一个步骤（下载安装包）

```bash
$ ls -a
.  ..  backups  downloads  .step

$ vim .step
{"state":"stop","step":5}
```

3）重新登录网站后台，切换到更新的位置，点击 `更新器`，（切记：不能点击之前失败页面中的`retry`按钮，否则会循环之前的错误），在新的界面中点击`continue`按钮，稍作等待即可顺利完成更新。

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220903173134.png)