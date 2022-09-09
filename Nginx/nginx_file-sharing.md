# Nginx File-sharing

> Nginx 简单搭建文件共享服务器

1）Nginx 安装过程略过，详细可参考 [Nginx Install(src)](nginx_install(src).md)

2）修改配置文件，开启 `alias` 功能

```bash
$ vim /usr/local/nginx/conf/nginx.conf
    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        # 文件共享功能
        location /tools {
            alias  /cloud_data;
            autoindex on;              # 开启目录浏览功能
            charset utf-8;             # 支持中文显示
            autoindex_exact_size off;  # 关闭详细文件大小统计，让文件大小显示MB，GB单位，默认为b；
            autoindex_localtime  on;   # 显示文件修改日期
        }
}
```

注：修改配置文件后，记得重启相关服务

3）页面美化(非必须)，首先查看已安装的模块

```bash
$ nginx -V
nginx version: nginx/1.21.6
built by gcc 9.3.1 20200408 (Red Hat 9.3.1-2) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/opt/nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
```

4）下载 [fancyindex](https://github.com/aperezdc/ngx-fancyindex/releases?utm_source=hacpai.com) ，重新配置 `configure`，添加 `--add-module=ngx-fancyindex-0.5.2`，其它不变

```bash
$ cd /usr/local/src/nginx-1.21.6
$ wget https://github.com/aperezdc/ngx-fancyindex/archive/refs/tags/v0.5.2.zip
$ unzip v0.5.2.zip
$ ./configure --prefix=/opt/nginx \
--with-compat \
--with-file-aio \
--with-threads \
--with-http_addition_module \
--with-http_auth_request_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-http_mp4_module \
--with-http_random_index_module \
--with-http_realip_module \
--with-http_secure_link_module \
--with-http_slice_module \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_sub_module \
--with-http_v2_module \
--with-mail \
--with-mail_ssl_module \
--with-stream \
--with-stream_realip_module \
--with-stream_ssl_module \
--with-stream_ssl_preread_module \
--with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' \
--with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' \
--add-module=ngx-fancyindex-0.5.2

$ make && make install
```

5）修改网站配置文件，再重启服务，具体效果如下图所示

```bash
        location ^~/tools {
            alias  /cloud_db/Software;
            charset utf-8;             # 支持中文显示

            autoindex on;              # 开启目录浏览功能
            autoindex_exact_size off;  # 关闭详细文件大小统计，让文件大小显示MB，GB单位，默认为b；
            autoindex_localtime  on;   # 显示文件修改日期
            
            fancyindex on;
            fancyindex_localtime on;
            fancyindex_exact_size off;
            fancyindex_default_sort date_desc;

        }
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220909164435.png)