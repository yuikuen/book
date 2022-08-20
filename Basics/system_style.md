# System Style

> PS1 是 Linux 终端用户的一个环境变量，用来定义命令行提示符的参数

1）了解 PS1

在终端输入命令，可得到当前 PS1 的定义值

```bash
$ echo $PS1
[\u@\h \W]\$
-------------------------------------------------------------------------------------
PS1='[\u@\h \W]\$ '  的意思就是：[当前用户的账号名称@主机名的第一个名字 工作目录的最后一层目录名]#
```

**PS1的常用参数以及含义:**

> - \e：控制符\033
> - \u：当前用户账号名称
> - \h：主机名称简称，仅取主机名中的第一个名字
> - \H：完整主机名
> - \v ：BASH 的版本信息
> - \w：当前工作目录，完整的工作目录名称
> - \W：当前工作目录基名，利用 basename 取得工作目录名称，只显示最后一个目录名
> - \d ：日期，格式为 weekday month date，例如："Mon Aug 1"
> - \t  ：24小时时间格式，如：HH：MM：SS
> - \T ：12小时时间格式，如：HH：MM
> - !    ：命令历史数
> - \#  ：开机后命令历史数，下达的第几个命令
> - \$  ：提示字符，如是 root 用户，提示符为 #，普通用户则为 $

2）颜色设置参数

在 PS1 中设置字符颜色的格式为：\[\e[F;Bm\]........\[\e[0m\]，其中 “F“ 为字体颜色，编号为 30-37，“B” 为背景颜色，编号为 40-47,\[\e[0m\]作为颜色设定的结束，只需将对应数字套入设置格式中即可

颜色对照表：

```
颜色对照表：
F    B
30   40   黑色
31   41   红色
32   42   绿色
33   43   黄色
34   44   蓝色
35   45   紫红色
36   46   青蓝色
37   47   白色
```

比如要设置命令行的格式为绿字黑底 (\[\e[32;40m\])，显示当前用户的账号名称 (\u)、主机的第一个名字 (\h)、完整的当前工作目录名称(\w)、24小时格式时间 (\t)，可以直接在命令行键入如下命令：

```bash
$ PS1='[\[\e[32;40m\]\u@\h \w \t]$ \[\e[0m\]'
```

3）永久保存命令行样式

上面的设置的作用域只有当前终端的登陆有效，关闭终端或退出登录即刻失效。要想永久性的保存设置，需要修改 .bashrc 配置文件

- **修改.bashrc文件**

```bash
$ vim ~/.bashrc
# 加入这一行
PS1="\[\e[37;40m\][\[\e[32;40m\]\u\[\e[37;40m\]@\h \[\e[36;40m\]\w\[\e[0m\]]\\$ "

$ source .bashrc
```

- **开机脚本启动**

```bash
$ echo 'PS1="\[\e[37;40m\][\[\e[32;40m\]\u\[\e[37;40m\]@\h \[\e[36;40m\]\w\[\e[0m\]]\\$ "' > /etc/profile.d/env.sh
```

*个人实例参考*，其中 [1;34;40m] 表示 [高亮;字体-蓝;背景-黑]

```bash
$ echo 'PS1="[\e[1;34;40m\]\d \e[31;40m\]\u\e[32;40m\]@\e[35;40m\]\h \e[36;40m\]\W\[\e[0m\]]\\$ "' > /etc/profile.d/env.sh
```