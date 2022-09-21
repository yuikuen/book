# Jenkins Install(YUM)

> YUM 安装 Jenkins

## 版本环境

- System：CentOS7.9.2009 Minimal
- Java：jdk-8u291-linux-x64
- Jenkins：Jenkins 最新版本

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装 JAVA

> 安装 Jenkins 前确保已配置好 JDK

```bash
$ wget https://download.oracle.com/otn/java/jdk/8u291-b10/d7fc238d0cbf4b0dac67be84580cfb4b/jdk-8u291-linux-x64.tar.gz
$ tar -xf jdk-8u291-linux-x64.tar.gz -C /usr/local/ && mv /usr/local/jdk1.8.0_291 /usr/local/java

$ vim /etc/profile
# java
JAVA_HOME=/usr/local/java
JRE_HOME=/usr/local/java/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
$ source /etc/profile
```

## 安装 Jenkins

安装 daemonize (按照官网提示 `yum install epel-release # repository that provides 'daemonize'`)

```bash
$ yum -y install epel-release
$ yum -y install daemonize
```

1）下载yum源并引入key（安装方法采用官网，下载较慢，不推荐使用）

```bash
$ wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
$ rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
$ yum install jenkins -y
```

2）修改 `/etc/sysconfig/jenkins`

```bash
$ vim /etc/sysconfig/jenkins
JENKINS_USER="root"
JENKINS_PORT="8080"
JENKINS_JAVA_OPTIONS="-Djava.awt.headless=true -Duser.timezone=Asia/Shanghai"
```

3）修改 `/etc/init.d/jenkins` 将 `java` 位置添加进 `candidates`

```bash
candidates="
/etc/alternatives/java
/usr/lib/jvm/java-1.8.0/bin/java
/usr/lib/jvm/jre-1.8.0/bin/java
/usr/lib/jvm/java-11.0/bin/java
/usr/lib/jvm/jre-11.0/bin/java
/usr/lib/jvm/java-11-openjdk-amd64
/usr/bin/java
# 添加自身安装的jdk的java位置
/usr/local/java/bin/java
"
```

4）启动服务查看密码进行安装

```bash
$ systemctl daemon-reload
$ systemctl enable --now jenkins
$ cat /var/lib/jenkins/secrets/initialAdminPassword
```