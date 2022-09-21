# Jenkins Install(WAR)

> War 安装 Jenkins-2.289.2

## 版本环境

- System：CentOS7.9.2009 Minimal
- Java：jdk-8u291-linux-x64
- Maven：apache-maven-3.8.1
- Jenkins：Jenkins-2.289.2

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

## 安装 Java

> 安装 Jenkins 前确保已配置好 [JDK](tps://www.oracle.com/cn/java/technologies/javase-downloads.html)

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

## 安装 Maven

```bash
$ wget https://mirrors.bfsu.edu.cn/apache/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
$ tar -xf apache-maven-3.8.1-bin.tar.gz -C /usr/local/ && mv /usr/local/apache-maven-3.8.1 /usr/local/maven
$ vim /etc/profile
# maven
export MAVEN_HOME=/usr/local/maven
export PATH=${JAVA_HOME}/bin:/usr/local/mysql/bin:${MAVEN_HOME}/bin:$PATH
$ source /etc/profile
```

## 安装 Jenkins

- 启动方式一：(Jar 包命令执行)

```bash
$ wget https://get.jenkins.io/war-stable/2.289.2/jenkins.war
$ nohup java -jar jenkins.war --httpPort=8080 &
```

- 启动方式二：(Tomcat 网站方式)

```bash
$ cd /usr/local/tomcat/webapps/ROOT
$ rm -rf *
$ jar –xvf jenkins.war && rm -rf jenkins.war
$ systemctl restart tomcat
```

根据提示获取密码，安装插件可选择跳过

```bash
$ cat /root/.jenkins/secrets/initialAdminPassword
cdd114654bf548f5a32df2bb79b5401b
```