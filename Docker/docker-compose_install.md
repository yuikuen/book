# Docker-Compose Install

> [Docker Compose](https://docs.docker.com/compose/install/) 是 docker 提供的一个命令行工具，用来定义和运行由多个容器组成的应用

```bash
$ curl -SL https://github.com/docker/compose/releases/download/v2.7.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
$ ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

注：不建议采用官网下载方式(国内访问较慢)，直接在 [GitHub](https://github.com/docker/compose/releases) 选择版本进行下载即可

```bash
$ wget https://github.com/docker/compose/releases/download/v2.10.0/docker-compose-linux-x86_64
$ mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
$ docker-compose --version
Docker Compose version v2.10.0
```