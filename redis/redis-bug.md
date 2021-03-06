# Reids异常问题处理集

> 安装后可通过 status 查看运行状态，根据实际异常错误进行处理

![image-20220627113931188](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220627113931188.png)

**问题1：**
`WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.`

解决方法：
```bash
# 将/proc/sys/net/core/somaxconn值设置为redis配置文件中的tcp-baklog值一致即可
$ echo '511' > /proc/sys/net/core/somaxconn

# 上述为临时处理方法，永久生效需要配置内核
$ echo 'net.core.somaxconn= 1024' >> /etc/sysctl.conf
$ sysctl -p
```

**问题2：**
`WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl...is to take effect.`

解决方法：
```bash
# 原因分析：overcommit_memory设置为0，在内存不足的情况下，后台保存会失败，要解决这个问题需要将此值改为1，然后重新加载，使其生效
$ echo 'vm.overcommit_memory=1' >> /etc/sysctl.conf 
$ sysctl -p
```

**问题3：**
`WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command .`

解决方法：
```bash
# 警告：您的内核中启用了透明的大页面（THP）支持。这将创建与ReDIS的延迟和内存使用问题。若要修复此问题，请运行命令“EngEng/mS/mL/mM/ExpListNo.HugPoIP/启用”为root，并将其添加到您的/etc/rc.local，以便在重新启动后保留设置。在禁用THP之后，必须重新启动redis。
$ echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled

# 上述为临时处理方法，永久生效需要配置到开机自启
$ vim /etc/rc.local
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
   echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
```

**问题4：**
`Warning: Could not create server TCP listening socket ::1:6379: bind: Cannot assign requested address`

解决方法：
```bash
# redis启动时载入的配置文件有一个无效参数，修改redis.conf，把bind 127.0.0.1的注释，并改成0
$ vim ./redis.conf
  72 # IF YOU ARE SURE YOU WANT YOUR INSTANCE TO LISTEN TO ALL THE INTERFACES
  73 # JUST COMMENT OUT THE FOLLOWING LINE.
  74 # ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  75 #bind 127.0.0.1 -::1
  76 bind 0.0.0.0
```