# Linux Command

> 本文仅记录常用或实用的命令及实例解析

## A

### awk

> 文本和数据进行处理的编程语言，用于在 linux/unix 下对文本和数据进行处理



## B

## C

## D

### du

> **du 命令**是对文件和目录磁盘使用的空间的查看

```bash
# 仅显示当前目录的总计值
$ du -sh .

# 文件从大到小排序
$ du -sh * | sort -rh
```

## E

## F

## G

## H

## I

## J

## K

## L

### ls

> 显示目录内容列表

```bash
# 列出所有文件（包括隐藏）的详细信息
$ ls -al
-rw-rw-r--  1 yuen yuen    0 May 18 11:14 1.jpg
-rw-r--r--. 1 yuen yuen  231 Apr  1  2020 .bashrc

# 列出详细信息并以可读大小显示文件大小
$ ls -hl
-rw-r--r--  1 root  root  8.3M Apr 27 09:24 apache-maven-3.8.5-bin.tar.gz
-rw-r--r--  1 root  root   20M Apr 22 11:07 erlang-23.3.4.11-1.el7.x86_64.rpm

# 按时间列出文件和文件夹详细信息
$ ls -lt
```

## M

### mv

> 用来对文件或目录重新命名

```bash
# 批量重命名文件名，递增方式
1.jpg  2.jpg  3.jpg  4.jpg  5.jpg  6.jpg  7.jpg  8.jpg  9.jpg
$ i=1;for x in *;do mv $x $i.jpg;let i=i+1;done && ls
10.jpg  11.jpg  12.jpg  13.jpg  14.jpg  15.jpg  16.jpg  17.jpg  18.jpg

# 无条件覆盖已存在的文件
$ mv -f *.txt /home/office

# 移动并排除指定目录
$ shop -s extglob
$ mv !(file1|file2) files
```

## N

## O

## P

## Q

## R

## S

### ss

> 比 netstat 好用的socket统计信息

```bash
# 查看进程使用的socket
$ ss -pl
Netid State      Recv-Q Send-Q                            Local Address:Port                                             Peer Address:Port                
nl    UNCONN     0      0                                          rtnl:kernel                                                       *                     
nl    UNCONN     0      0                                          rtnl:NetworkManager/902                                           *                     
nl    UNCONN     768    0                                          rtnl:dockerd/1635                                                 *     

# 找出打开套接字/端口应用程序
$ ss -lp | grep 3306
tcp    LISTEN     0      80       [::]:3306                   [::]:*

# 只显示 unix 连接
$ ss -x
Netid State      Recv-Q Send-Q                            Local Address:Port                                             Peer Address:Port                
u_dgr ESTAB      0      0                           /run/systemd/notify 12594                                                       * 0                    
u_dgr ESTAB      0      0                   /run/systemd/journal/socket 12601                                                       * 0 
```

## T

### touch

> 创建新的空文件

```bash
# 批量创建文件
$ touch file{1..5}.txt

# 创建 job1.md 文件，并写入 job 1
$ echo "job 1" > job1.md
```

## U

## V

## W

## X

## Y

## Z
