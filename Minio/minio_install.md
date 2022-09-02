# MinIO Install

> 一款基于Go语言发开的高性能、分布式的对象存储系统

## 单机模式

1）创建目录并下载授权

```bash
$ mkdir -p /opt/minio/{data,logs}
$ cd /opt/minio
$ wget https://dl.min.io/server/minio/release/linux-amd64/minio
$ chmod +x minio
```

2）设置用户密码

> 如未定义，则默认的用户密码均为 minioadmin

```bash
$ cat /etc/profile.d/minio.sh 
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=Admin@123

$ source /etc/profile
```

3）后台启动，指定日志路径

```bash
$ nohup ./minio server --address ':9000' --console-address ':9001' ./data/ > ./logs/minio.log 2>&1 &
```
此处 `--address ':9000'` 指的是 api 端口，

注：如未指定端口，api 默认端口为 9000，console 端口则自动随机分配

```bash
$ nohup ./minio server ./data/ > ./logs/minio.log 2>&1
$ cat minio.log 
nohup: ignoring input
MinIO Object Storage Server
Copyright: 2015-2022 MinIO, Inc.
License: GNU AGPLv3 <https://www.gnu.org/licenses/agpl-3.0.html>
Version: RELEASE.2022-08-26T19-53-15Z (go1.18.5 linux/amd64)

Status:         1 Online, 0 Offline. 
API: http://188.188.4.206:9000  http://127.0.0.1:9000     
Console: http://188.188.4.206:42065 http://127.0.0.1:42065   

Documentation: https://docs.min.io
```

## 脚本方式

1）创建目录并下载授权

```bash
$ mkdir -p /opt/minio/{app,config,bin,data,logs}
$ cd /opt/minio/app
$ wget https://dl.min.io/server/minio/release/linux-amd64/minio
$ chmod +x minio
```

2）创建部署启动脚本

```bash
$ touch /opt/minio/bin/run.sh && vim !$
#!/bin/bash
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=Admin@123
/opt/minio/app/minio server --config-dir /opt/minio/config --address ":9000" --console-address ":9001" /opt/minio/data >/opt/minio/logs/minio.log
$ chmod +x /opt/minio/bin/run.sh
```

3）创建服务文件并设置开机自启

```bash
$ cat /usr/lib/systemd/system/minio.service
[Unit]
Description=Minio service
Documentation=https://docs.minio.io/

[Service]
ExecStart=/opt/minio/bin/run.sh
# 表示任务异常退出都会重启
Restart=on-failure
# 表示重启等待时间为5s
RestartSec=5

[Install]
WantedBy=multi-user.target

$ chmod +x !$
$ systemctl enable --now minio && systemctl status minio
● minio.service - Minio service
   Loaded: loaded (/usr/lib/systemd/system/minio.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-08-30 14:55:02 CST; 17min ago
     Docs: https://docs.minio.io/
 Main PID: 895 (run.sh)
   CGroup: /system.slice/minio.service
           ├─895 /bin/bash /opt/minio/bin/run.sh
           └─901 /opt/minio/app/minio server --config-dir /opt/minio/config --address :9000 --console-address :9001 /opt/minio/data

Aug 30 14:55:02 mw-06 systemd[1]: Started Minio service.
```

## 单机容器

> Docker 单机服务端模式部署，未使用纠错码

```bash
$ mkdir -p /opt/minio/{cert,data,config}
$ docker run -it -d --name minio \
--restart=always \
--net host \
-e "MINIO_ROOT_USER=admin" \
-e "MINIO_ROOT_PASSWORD=Admin@123" \
-v /etc/localtime:/etc/localtime \
-v $PWD/data:/data \
-v $PWD/config:/root/.minio \
minio/minio server /data --address ":9000" --console-address ":9001"
```

> Docker-Compose 编排单机服务端模式部署

```docker
version: '3.7'
services:
  minio:
    image: minio/minio:RELEASE.2022-08-26T19-53-15Z
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    restart: always
    command: server /data --address ":9000" --console-address ":9001"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: Admin@123 
    logging:
      options:
        max-size: "50M" # 最大文件上传限制
        max-file: "10"
      driver: json-file
    volumes:
      - $PWD:/data # 映射文件路径
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone/timezone:/etc/timezone:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
```

> Docker-Compose 编排单机多容器+负载均衡，可参考 [官方链接](https://raw.githubusercontent.com/minio/minio/master/docs/orchestration/docker-compose/docker-compose.yaml)

```docker
version: '3.7'

# 所有容器通用的设置和配置
x-minio-common: &minio-common
  image: minio/minio
  command: server --console-address ":9001" http://minio{1...4}/data
  expose:
    - "9000"
  # environment:
    # MINIO_ROOT_USER: minioadmin
    # MINIO_ROOT_PASSWORD: minioadmin
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
    interval: 30s
    timeout: 20s
    retries: 3

# 启动4个docker容器运行minio服务器实例
# 使用nginx反向代理9000端口，负载均衡, 你可以通过9001、9002、9003、9004端口访问它们的web console
services:
  minio1:
    <<: *minio-common
    hostname: minio1
    ports:
      - "9001:9001"
    volumes:
      - ./data1:/data

  minio2:
    <<: *minio-common
    hostname: minio2
    ports:
      - "9002:9001"
    volumes:
      - ./data2:/data

  minio3:
    <<: *minio-common
    hostname: minio3
    ports:
      - "9003:9001"
    volumes:
      - ./data3:/data

  minio4:
    <<: *minio-common
    hostname: minio4
    ports:
      - "9004:9001"
    volumes:
      - ./data4:/data

  nginx:
    image: nginx:1.19.2-alpine
    hostname: nginx
    volumes:
      - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "9000:9000"
    depends_on:
      - minio1
      - minio2
      - minio3
      - minio4
```

```sh
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  4096;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;

    # include /etc/nginx/conf.d/*.conf;

    upstream minio {
        server minio1:9000;
        server minio2:9000;
        server minio3:9000;
        server minio4:9000;
    }

    server {
        listen       9000;
        listen  [::]:9000;
        server_name  localhost;

        # To allow special characters in headers
        ignore_invalid_headers off;
        # Allow any size file to be uploaded.
        # Set to a value such as 1000m; to restrict file size to a specific value
        client_max_body_size 0;
        # To disable buffering
        proxy_buffering off;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 300;
            # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            chunked_transfer_encoding off;

            proxy_pass http://minio;
        }
    }

}

```