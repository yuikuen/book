# Dockerfile JDK

> Dockerfile 构建 JDK 镜像

## 简要说明

Dockerfile 是一个文本文档，文本内包含许多命令，每一条命令构建一层，因此每一条指令的内容，就是描述该层应当如何构建；

一般 Docker 中的镜像都可以在官方 [DockerHub](https://registry.hub.docker.com/) 中进行下载，如需自定义镜像，建议选用 `alpine & slim` 作为使用；

## 基本结构

Dockerfile 由一行行命令语句组成，并支持以 # 开头的注释行

[Dockerfile 参数详解]:https://docs.docker.com/engine/reference/builder/

**主体内容分为四部分：**

| 主体内容           | 命令       | 备注                                                         |
| ------------------ | ---------- | ------------------------------------------------------------ |
| 基础镜像信息       | FROM       | 指令用于指定要构建的镜像的基础镜像。它通常是 Dockerfile 中的第一条指令。 |
| 维护者信息         | MAINTAINER | 指令设置生成图像的作者字段                                   |
| 镜像操作指令       | RUN        | 指令用于在镜像中执行命令，这会创建新的镜像层。每个 RUN 指令创建一个新的镜像层 |
|                    | (COPY/ADD) | 指令用于将文件作为一个新的层添加到镜像中。使用 COPY 指令将应用代码赋值到镜像中 |
|                    | EXPOSE     | 指令用于记录应用所使用的网络端口                             |
|                    | WORKDIR    | 在执行 RUN 后面的 shell 命令前会先 cd 进 WORKDIR 后面的目录  |
|                    | ONBUILD    | 指令将映像用作另一个构建的基础时，将在以后的时间向该映像添加触发指令 |
| 容器启动时执行指令 | CMD        | 指令中只能有一条指令 Dockerfile，如果列出多个，则只有最后一个 CMD 才会生效,指令的主要目的是为执行中的容器提供默认值 |
|                    | ENTRYPOINT | 指令用于指定镜像以容器方式启动后默认运行的程序               |

--------------

## 前期准备

[所有 JDK 下载地址]:https://www.oracle.com/java/technologies/oracle-java-archive-downloads.html

2019年4月16日，Oracle 发布了新的 JDK 8 的更新，版本号为 8u211 和 8u212。与以往不同的是，新版本的许可协议从BCL换成了OTN，这就意味着不能在生产环境使用这个版本了，因此现选用的是 Oracle JDK 最后的免费版本 8u202。

另外由于默认 JDK8 是不能使用高强度加密算法的， 需要把 `jce_policy-8.zip` 中的两个 jar 包拷贝到 `\lib\security` 下面， 替换掉两个原有的同名文件，以开启对高强度加密算法支持

```bash
$ mkdir -p /usr/local/src/dockerjdk8 && cd !$
$ wget https://www.oracle.com/webapps/redirect/signon?nexturl=https://download.oracle.com/otn/java/jdk/8u202-b08/1961070e4c9b4e26a04e7f5a083f551e/jdk-8u202-linux-x64.tar.gz
$ wget https://download.oracle.com/otn-pub/java/jce/8/jce_policy-8.zip
```

## 镜像构建

[Github 参考链接-glibc]:https://github.com/Docker-Hub-frolvlad/docker-alpine-glibc
[Github 参考链接-java]:https://github.com/Docker-Hub-frolvlad/docker-alpine-java

构建选用的是 alpine 镜像，alpine 表明镜像的操作系统是 alpine linux，alpine linux本身很小，镜像的大小是5M左右，而运行 JDK 需要 Glibc 库的依赖。所以先基于 Alpine Linux 镜像，重新构建包含 glibc 库，从而再根据 alpine-glibc 镜像来构建 JDK-alpine-glibc

```dockerfile
$ vim Dockerfile-glibc
FROM alpine:3.14

ENV LANG=C.UTF-8

# Here we install GNU libc (aka glibc) and set C.UTF-8 locale as default.
RUN echo http://mirrors.ustc.edu.cn/alpine/v3.10/main > /etc/apk/repositories && \
    echo http://mirrors.ustc.edu.cn/alpine/v3.10/community >> /etc/apk/repositories && \
    ALPINE_GLIBC_BASE_URL="https://github.com/sgerrand/alpine-pkg-glibc/releases/download" && \
    ALPINE_GLIBC_PACKAGE_VERSION="2.33-r0" && \
    ALPINE_GLIBC_BASE_PACKAGE_FILENAME="glibc-$ALPINE_GLIBC_PACKAGE_VERSION.apk" && \
    ALPINE_GLIBC_BIN_PACKAGE_FILENAME="glibc-bin-$ALPINE_GLIBC_PACKAGE_VERSION.apk" && \
    ALPINE_GLIBC_I18N_PACKAGE_FILENAME="glibc-i18n-$ALPINE_GLIBC_PACKAGE_VERSION.apk" && \
    apk add --no-cache --virtual=.build-dependencies wget ca-certificates && \
    echo \
        "-----BEGIN PUBLIC KEY-----\
        MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEApZ2u1KJKUu/fW4A25y9m\
        y70AGEa/J3Wi5ibNVGNn1gT1r0VfgeWd0pUybS4UmcHdiNzxJPgoWQhV2SSW1JYu\
        tOqKZF5QSN6X937PTUpNBjUvLtTQ1ve1fp39uf/lEXPpFpOPL88LKnDBgbh7wkCp\
        m2KzLVGChf83MS0ShL6G9EQIAUxLm99VpgRjwqTQ/KfzGtpke1wqws4au0Ab4qPY\
        KXvMLSPLUp7cfulWvhmZSegr5AdhNw5KNizPqCJT8ZrGvgHypXyiFvvAH5YRtSsc\
        Zvo9GI2e2MaZyo9/lvb+LbLEJZKEQckqRj4P26gmASrZEPStwc+yqy1ShHLA0j6m\
        1QIDAQAB\
        -----END PUBLIC KEY-----" | sed 's/   */\n/g' > "/etc/apk/keys/sgerrand.rsa.pub" && \
    wget \
        "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_I18N_PACKAGE_FILENAME" && \
    apk add --no-cache \
        "$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_I18N_PACKAGE_FILENAME" && \
    \
    rm "/etc/apk/keys/sgerrand.rsa.pub" && \
    /usr/glibc-compat/bin/localedef --force --inputfile POSIX --charmap UTF-8 "$LANG" || true && \
    echo "export LANG=$LANG" > /etc/profile.d/locale.sh && \
    \
    apk del glibc-i18n && \
    \
    rm "/root/.wget-hsts" && \
    apk del .build-dependencies && \
    rm \
        "$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_I18N_PACKAGE_FILENAME"
        
$ docker build -f Dockerfile-glibc -t alpine-glibc:a3.14_g2.33 .

$ docker images
REPOSITORY                      TAG                      IMAGE ID       CREATED         SIZE
alpine-glibc                    3.14                     4fa28b64d76b   5 seconds ago   17.7MB
```

```dockerfile
$ vim Dockerfile-jdk
FROM alpine-glibc:a3.14_g2.33

ENV JAVA_VERSION=8 \
    JAVA_UPDATE=202 \
    JAVA_BUILD=08 \
    JAVA_HOME="/usr/lib/jvm/default-jvm"

ADD jdk-8u202-linux-x64.tar.gz /tmp
COPY jce_policy-8.zip /tmp

RUN apk add --no-cache --virtual=build-dependencies wget ca-certificates unzip && \
    cd "/tmp" && \
    mkdir -p "/usr/lib/jvm" && \
    mv "/tmp/jdk1.${JAVA_VERSION}.0_${JAVA_UPDATE}" "/usr/lib/jvm/java-${JAVA_VERSION}-oracle" && \
    ln -s "java-${JAVA_VERSION}-oracle" "$JAVA_HOME" && \
    ln -s "$JAVA_HOME/bin/"* "/usr/bin/" && \
    rm -rf "$JAVA_HOME/"*src.zip && \
    rm -rf "$JAVA_HOME/lib/missioncontrol" \
           "$JAVA_HOME/lib/visualvm" \
           "$JAVA_HOME/lib/"*javafx* \
           "$JAVA_HOME/jre/lib/plugin.jar" \
           "$JAVA_HOME/jre/lib/ext/jfxrt.jar" \
           "$JAVA_HOME/jre/bin/javaws" \
           "$JAVA_HOME/jre/lib/javaws.jar" \
           "$JAVA_HOME/jre/lib/desktop" \
           "$JAVA_HOME/jre/plugin" \
           "$JAVA_HOME/jre/lib/"deploy* \
           "$JAVA_HOME/jre/lib/"*javafx* \
           "$JAVA_HOME/jre/lib/"*jfx* \
           "$JAVA_HOME/jre/lib/amd64/libdecora_sse.so" \
           "$JAVA_HOME/jre/lib/amd64/"libprism_*.so \
           "$JAVA_HOME/jre/lib/amd64/libfxplugins.so" \
           "$JAVA_HOME/jre/lib/amd64/libglass.so" \
           "$JAVA_HOME/jre/lib/amd64/libgstreamer-lite.so" \
           "$JAVA_HOME/jre/lib/amd64/"libjavafx*.so \
           "$JAVA_HOME/jre/lib/amd64/"libjfx*.so && \
    rm -rf "$JAVA_HOME/jre/bin/jjs" \
           "$JAVA_HOME/jre/bin/keytool" \
           "$JAVA_HOME/jre/bin/orbd" \
           "$JAVA_HOME/jre/bin/pack200" \
           "$JAVA_HOME/jre/bin/policytool" \
           "$JAVA_HOME/jre/bin/rmid" \
           "$JAVA_HOME/jre/bin/rmiregistry" \
           "$JAVA_HOME/jre/bin/servertool" \
           "$JAVA_HOME/jre/bin/tnameserv" \
           "$JAVA_HOME/jre/bin/unpack200" \
           "$JAVA_HOME/jre/lib/ext/nashorn.jar" \
           "$JAVA_HOME/jre/lib/jfr.jar" \
           "$JAVA_HOME/jre/lib/jfr" \
           "$JAVA_HOME/jre/lib/oblique-fonts" && \
    unzip -jo -d "${JAVA_HOME}/jre/lib/security" "jce_policy-${JAVA_VERSION}.zip" && \
    rm "${JAVA_HOME}/jre/lib/security/README.txt" && \
    apk del build-dependencies && \
    rm "/tmp/"* && \
    apk add tzdata && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo 'public class Main { public static void main(String[] args) { System.out.println("Java code is running fine!"); } }' > Main.java && \
    javac Main.java && \
    java Main && \
    rm -r "/tmp/"*
    
$ docker build -f Dockerfile-jdk -t jdk:alpine_glibc-8u202 .
```

## 验证测试

```bash
$ docker images
REPOSITORY                      TAG                       IMAGE ID       CREATED          SIZE
yk-harbor.net/public/jdk        alpine_glibc-8u202        82403b887e0a   9 days ago       168MB

$ docker run -it --rm yk-harbor.net/public/jdk:alpine_glibc-8u202 /bin/sh
/ # java -version
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)
```
