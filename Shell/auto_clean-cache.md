# Auto Clean-Cache

## 示例 1

```bash
#!/bin/bash

used=`free -m | awk 'NR==2' | awk '{print $3}'`
free=`free -m | awk 'NR==2' | awk '{print $4}'`
echo "===========================" >> /var/cache/memory.log
date >> /var/cache/memory.log
echo "Memory usage before | [Use：${used}MB][Free：${free}MB]" >> /var/cache/memory.log
if [ $free -le 1000 ] ; then
                sync && echo 1 > /proc/sys/vm/drop_caches
                sync && echo 2 > /proc/sys/vm/drop_caches
                sync && echo 3 > /proc/sys/vm/drop_caches
                used_ok=`free -m | awk 'NR==2' | awk '{print $3}'`
                free_ok=`free -m | awk 'NR==2' | awk '{print $4}'`
                echo "Memory usage after | [Use：${used_ok}MB][Free：${free_ok}MB]" >> /var/cache/memory.log
                echo "OK" >> /var/cache/memory.log
else
                echo "Not required" >> /var/cache/memory.log
fi
exit 1
```

## 示例 2

```bash
#!/bin/bash

DAY=`date +"%Y-%m-%d %H:%M"`
MEM=`free -m | grep -i "mem" | awk '{print $4}'`

if [ $MEM -ge 1000 ];then
  echo "$DAY MEM:$MEM No problem" >> Memory.log
else
  sync; echo 3 > /proc/sys/vm/drop_caches
  echo "$DAY MEM:$MEM Deleted Successful" >> Memory.log
fi
```