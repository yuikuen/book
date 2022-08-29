# Java Install(SRC)

> 源码编译安装 Java 1.8.x

## 版本环境

Java 安装一般为 [OpenJDK](https://jdk.java.net/8/) 或 [Oracle Java](https://www.oracle.com/java/technologies/downloads/)，按需选择

- System：CentOS7.9.2009 Minimal
- Java：jdk-8u333-linux-x64

```bash
# 查看
rpm -qa | grep java
rpm -qa | grep jdk
# 批量卸载
rpm -qa | grep jdk | xargs rpm -e --nodeps
rpm -qa | grep java | xargs rpm -e --nodeps
```

## Oracle Java

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

$ ln -s /usr/local/java/bin/java /usr/bin/java
```

