# Nexus2 Install(SRC)

## 版本环境

- System：CentOS7.9.2009 Minimal
- Java：jdk-8u291-linux-x64
- Maven：apache-maven-3.8.1-bin
- Nexus：nexus-2.14.20-02-bundle

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

**首先安装好 Jdk 和 Maven 环境**

## 安装 Java

```bash
$ wget https://download.oracle.com/otn/java/jdk/8u291-b10/d7fc238d0cbf4b0dac67be84580cfb4b/jdk-8u291-linux-x64.tar.gz
$ tar -xf jdk-8u291-linux-x64.tar.gz -C /usr/local/ 
$ mv /usr/local/jdk1.8.0_291 /usr/local/java

$ vim /etc/profile
# java
JAVA_HOME=/usr/local/java
JRE_HOME=/usr/local/java/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH

$ source /etc/profile
```

## 安装 Maven 

```bash
$ wget https://mirrors.bfsu.edu.cn/apache/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
$ tar -xf apache-maven-3.8.1-bin.tar.gz -C /usr/local/ 
$ mv /usr/local/apache-maven-3.8.1 /usr/local/maven

$ vim /etc/profile
# maven
export MAVEN_HOME=/usr/local/maven
export PATH=${JAVA_HOME}/bin:/usr/local/mysql/bin:${MAVEN_HOME}/bin:$PATH
$ source /etc/profile
```

## 安装 Nexus2

```bash
$ wget https://download.sonatype.com/nexus/oss/nexus-2.14.20-02-bundle.tar.gz
$ mkdir /usr/local/nexus
$ tar -xf nexus-2.14.20-02-bundle.tar.gz -C /usr/local/nexus
```

修改/nexus-2.14.20-02/conf 目录下：nexus.properties，端口可以自定义，避免冲突

```bash
# Sonatype Nexus
# ==============
# This is the most basic configuration of Nexus.

# Jetty section
application-port=8081
application-host=0.0.0.0
nexus-webapp=${bundleBasedir}/nexus
nexus-webapp-context-path=/nexus

# Nexus section
nexus-work=${bundleBasedir}/../sonatype-work/nexus
runtime=${bundleBasedir}/nexus/WEB-INF

# orientdb buffer size in megabytes
storage.diskCache.bufferSize=4096
```

在 `/nexus-2.14.20-02/bin/`目录下执行：`RUN_AS_USER=root ./nexus start`

访问 [http://ip:port/nexus](http://192.168.100.52:8081/nexus) ，登陆Nexus，默认的用户名是：admin、密码是：admin123

**设置一个环境变量，让 nexus 直接用 root 用户启动**

```bash
# 配置环境变量
$ echo >> /etc/profile
$ echo "# set nexus run as root" >> /etc/profile
$ echo "export RUN_AS_USER=root" >> /etc/profile

# 使配置文件生效
$ source /etc/profile

$ /opt/nexus/nexus-2.14.20-02/bin/nexus start
****************************************
WARNING - NOT RECOMMENDED TO RUN AS ROOT
****************************************
Starting Nexus OSS...
Started Nexus OSS.

$ netstat -tulnp | grep $(jps -l | grep org.sonatype.nexus.bootstrap.jsw.JswLauncher | awk '{print $1}')
tcp        0      0 0.0.0.0:8081            0.0.0.0:*               LISTEN      1768/java           
tcp        0      0 127.0.0.1:32000         0.0.0.0:*               LISTEN      1768/java
```

## 开机自启

```bash
$ cat > /lib/systemd/system/nexus.service <<-EOF
[Unit]
Description=nexus
After=network.target
[Service]
Type=forking
Environment=RUN_AS_USER=root
Environment=PATH=/root/.tiup/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/local/java/bin:/usr/local/maven/bin:/root/bin:/usr/local/java/bin:/usr/local/maven/bin:/usr/local/java/bin:/usr/local/maven/bin
ExecStart=/opt/nexus/nexus-2.14.20-02/bin/nexus start
ExecReload=/opt/nexus/nexus-2.14.20-02/bin/nexus restart
ExecStop=/opt/nexus/nexus-2.14.20-02/bin/nexus stop
PrivateTmp=true
[Install]
WantedBy=multi-user.target
EOF

$ lsof -i:8081
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    3299 root  491u  IPv4  38418      0t0  TCP *:tproxy (LISTEN)
java    3299 root  495u  IPv4  38420      0t0  TCP data:tproxy->ec2-100-20-0-71.us-west-2.compute.amazonaws.com:57067 (CLOSE_WAIT)
java    3299 root  496u  IPv4  38470      0t0  TCP data:tproxy->ec2-100-20-0-71.us-west-2.compute.amazonaws.com:62446 (CLOSE_WAIT)
java    3299 root  497u  IPv4  38471      0t0  TCP data:tproxy->ec2-100-20-0-71.us-west-2.compute.amazonaws.com:61221 (CLOSE_WAIT)
java    3299 root  498u  IPv4  38472      0t0  TCP data:tproxy->ec2-100-20-0-71.us-west-2.compute.amazonaws.com:62283 (ESTABLISHED)
java    3299 root  500u  IPv4  38473      0t0  TCP data:tproxy->ec2-100-20-0-71.us-west-2.compute.amazonaws.com:58709 (ESTABLISHED)
java    3299 root  501u  IPv4  38474      0t0  TCP data:tproxy->ec2-100-20-0-71.us-west-2.compute.amazonaws.com:57747 (ESTABLISHED)
```

## 基本配置

修改密码

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20210803152414668.png)

点击`Repository Targets`可以管理仓库

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20210803152624146.png)

**配置 maven 代理仓库**

默认的 maven 远程仓库地址为：https://repo1.maven.org/maven2/ 地址在美国，太慢了，所以我们使用阿里云的远程仓库

```xml
http://maven.aliyun.com/nexus/content/groups/public/
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20210803153135248.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20210803153827456.png)

更改 maven 当中的 settings.xml 配置文件，将中央仓库改为以下值（`注意IP地址改为自己的`）：

```bash
<mirror>
  <id>central</id>
  <name>my maven</name>
  <url>http://188.188.3.110:8081/nexus/content/repositories/central/</url>
  <mirrorOf>central</mirrorOf>
</mirror>
```

在`<mirror> </mirror>`注释掉原有仓库，添加新的仓库，再打包 maven 项目即使用的正是搭建的 maven 私服