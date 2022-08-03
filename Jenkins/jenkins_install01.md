# Jenkins Install(RPM)

> RPM 安装 Jenkins 2.346.x

## 版本环境

Jenkins 2.357 和将发布的 LTS 版本开始，Jenkins 需要 Java 11 才能使用，将放弃 Java 8，请注意兼容性问题。

- System：CentOS7.9.2009 Minimal
- Java：jdk-8u333-linux-x64
- Jenkins：jenkins-2.346.2-1.1

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

```bash
# 按照官网提示，提前安装daemonize 
$ yum -y install epel-release daemonize
```

## 安装 Java

1）自行到 [ORACLE](https://www.oracle.com/java/technologies/downloads/) 下载 Java 包
```bash
$ tar -xf jdk-8u333-linux-x64.tar.gz -C /usr/local/
$ mv /usr/local/jdk1.8.0_333 /usr/local/java
```

2）配置系统环境变量

```bash
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

## 安装 Jenkins

> [官方下载](https://archives.jenkins-ci.org/redhat-stable/) 较慢，建议选择 [国内源](https://mirrors.tuna.tsinghua.edu.cn/jenkins/redhat-stable/) 进行下载，安装过程可参考 [官网](tps://www.jenkins.io/zh/doc/book/installing)

1）下载后直接安装

```bash
$ wget https://mirrors.tuna.tsinghua.edu.cn/jenkins/redhat-stable/jenkins-2.346.2-1.1.noarch.rpm --no-check-certificate
$ rpm -ivh jenkins-2.346.2-1.1.noarch.rpm --force --nodeps
warning: jenkins-2.346.2-1.1.noarch.rpm: Header V4 RSA/SHA512 Signature, key ID 45f2c3d5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:jenkins-2.346.2-1.1              ################################# [100%]
```

2）修改配置文件的默认参数

```bash
$ cat /etc/sysconfig/jenkins | grep -v '#' | grep -v '^$'
# 默认即可，此处只添加了时区
JENKINS_HOME="/var/lib/jenkins"
JENKINS_JAVA_CMD=""
JENKINS_USER="jenkins"
JENKINS_JAVA_OPTIONS="-Djava.awt.headless=true -Duser.timezone=Asia/Shanghai"
JENKINS_PORT="8080"
JENKINS_LISTEN_ADDRESS=""
JENKINS_HTTPS_PORT=""
JENKINS_HTTPS_KEYSTORE=""
JENKINS_HTTPS_KEYSTORE_PASSWORD=""
JENKINS_HTTPS_LISTEN_ADDRESS=""
JENKINS_HTTP2_PORT=""
JENKINS_HTTP2_LISTEN_ADDRESS=""
JENKINS_DEBUG_LEVEL="5"
JENKINS_ENABLE_ACCESS_LOG="no"
JENKINS_HANDLER_MAX="100"
JENKINS_HANDLER_IDLE="20"
JENKINS_EXTRA_LIB_FOLDER=""
JENKINS_ARGS=""
```

3）修改启动的设置，增加 Java 文件目录路径

```bash
$ vim /etc/init.d/jenkins +97
# Search usable Java as /usr/bin/java might not point to minimal version required by Jenkins.
# see http://www.nabble.com/guinea-pigs-wanted-----Hudson-RPM-for-RedHat-Linux-td25673707.html
candidates="
/etc/alternatives/java
/usr/lib/jvm/java-1.8.0/bin/java
/usr/lib/jvm/jre-1.8.0/bin/java
/usr/lib/jvm/java-11.0/bin/java
/usr/lib/jvm/jre-11.0/bin/java
/usr/lib/jvm/java-11-openjdk-amd64
/usr/bin/java
# 添加下述java的安装目录
/usr/local/java/bin/java
"
```

4）启动程序

```bash
$ systemctl daemon-reload
$ systemctl enable --now jenkins
$ systemctl status jenkins
● jenkins.service - Jenkins Continuous Integration Server
   Loaded: loaded (/usr/lib/systemd/system/jenkins.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-08-02 15:09:53 CST; 1min 27s ago
 Main PID: 1714 (java)
   CGroup: /system.slice/jenkins.service
           └─1714 /usr/bin/java -Djava.awt.headless=true -jar /usr/share/java/jenkins.war --webroot=%C/jenkins/war --httpPort=8080

Aug 02 15:08:42 ssd-dev03 jenkins[1714]: Jenkins initial setup is required. An admin user has been created and a password generated.
Aug 02 15:08:42 ssd-dev03 jenkins[1714]: Please use the following password to proceed to installation:
Aug 02 15:08:42 ssd-dev03 jenkins[1714]: b63be81c21f84b0195703e49a0fff5b2
Aug 02 15:08:42 ssd-dev03 jenkins[1714]: This may also be found at: /var/lib/jenkins/secrets/initialAdminPassword
Aug 02 15:08:42 ssd-dev03 jenkins[1714]: *************************************************************
Aug 02 15:08:42 ssd-dev03 jenkins[1714]: *************************************************************
Aug 02 15:08:42 ssd-dev03 jenkins[1714]: *************************************************************
Aug 02 15:09:53 ssd-dev03 jenkins[1714]: 2022-08-02 07:09:53.214+0000 [id=31]        INFO        jenkins.InitReactorRunner$1#onAttained: Completed initialization
Aug 02 15:09:53 ssd-dev03 jenkins[1714]: 2022-08-02 07:09:53.224+0000 [id=21]        INFO        hudson.lifecycle.Lifecycle#onReady: Jenkins is fully up and running
Aug 02 15:09:53 ssd-dev03 systemd[1]: Started Jenkins Continuous Integration Server.
```

**如上述方法未成功并且如下错误，可按下面操作**

`Job for jenkins.service failed because the control process exited with error code. See "systemctl status jenkins.service" and "journalctl -xe" for details.`

```bash
$ ln -s /usr/local/java/bin/java /usr/bin/java
```

5）打开浏览器 http://jenkins-server:8080，按上述提示默认跳过安装即可

> 建议不选择插件直接跳过安装

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220802151524.png)