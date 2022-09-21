# Zookeeper Install(SRC)

> Zookeeper 二进制集群部署

[Zookeeper](http://zookeeper.apache.org) 是一个开源的分布式协调服务，一个典型的分布式数据一致性解决方案，分布式应用程序可以基于它实现数据的发布/订阅、负载均衡、名称服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列

## 版本环境

Jenkins 2.357 和将发布的 LTS 版本开始，Jenkins 需要 Java 11 才能使用，将放弃 Java 8，请注意兼容性问题。

- System：CentOS7.9.2009 Minimal
- Java：java-11-openjdk-devel
- Zookeeper：apache-zookeeper-3.7.0-bin

```bash
# 演示环境，直接关闭 SELinux & Firewalld
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config && setenforce 0 
$ systemctl disable --now firewalld.service 
```

**集群节点规划**

|HostName|IP|角色|
|--|--|--|
|zookeeper01|188.188.4.141|zookeeper节点1|
|zookeeper02|188.188.4.142|zookeeper节点2|
|zookeeper03|188.188.4.143|zookeeper节点3|





