# Nexus3 Install

## 版本环境

- System：CentOS7.9.2009 Minimal
- Java：
- Nexus：nexus-3.41.1-01-unix

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装 Jdk

```bash
$ tar -xf jdk-8u333-linux-x64.tar.gz -C /usr/local/
$ mv /usr/local/jdk1.8.0_333 /usr/local/java

$ cat /etc/profile.d/java8.sh
JAVA_HOME=/usr/local/java
JRE_HOME=/usr/local/java/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH

$ source /etc/profile
$ java -version
java version "1.8.0_333"
Java(TM) SE Runtime Environment (build 1.8.0_333-b02)
Java HotSpot(TM) 64-Bit Server VM (build 25.333-b02, mixed mode)
```

## 安装 Nexus

> 官网下载 [Nexus3](https://help.sonatype.com/repomanager3/product-information/download)，注意可能需要梯子

1）创建用户

```bash
$ groupadd nexus
$ useradd -g nexus nexus
$ groups nexus
nexus : nexus
```

2）解压程序并修改目录权限

```bash
$ mkdir -p /opt/nexus
$ tar -xf nexus-3.41.1-01-unix.tar.gz -C /opt/nexus
$ chown -R nexus. nexus/
```

3）配置文件修改

- 启动用户

```bash
$ vim ./nexus/nexus-3.41.1-01/bin/nexus.rc
run_as_user="nexus"
```

- JDK 版本信息

```bash
$ vim ./nexus/nexus-3.41.1-01/bin/nexus
INSTALL4J_JAVA_HOME_OVERRIDE=/usr/local/java
```

- 默认端口

```bash
$ vim ./nexus/nexus-3.41.1-01/etc/nexus-default.properties
# Jetty section
application-port=8081
```

4）创建启动脚本

```bash
$ vim /usr/lib/systemd/system/nexus.service
[Unit]
Description=nexus
After=network.target

[Service]
Type=forking
ExecStart=/opt/nexus/nexus-3.41.1-01/bin/nexus start
ExecReload=/opt/nexus/nexus-3.41.1-01/bin/nexus restart
ExecStop=/opt/nexus/nexus-3.41.1-01/bin/nexus stop
User=nexus
Restart=on-abort
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

5）启动服务并按提示查看密码进行登录修改

```bash
$ systemctl enable --now nexus && systemctl status nexus
$ cat /opt/nexus/sonatype-work/nexus3/admin.password
705683b1-0244-4adb-bf63-7fc3436aa6cb
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826111856.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220826112305.png)