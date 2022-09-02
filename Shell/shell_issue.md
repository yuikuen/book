# Shell Issue

> 主要记录一些小技巧

## bash/sh 脚本调试

使用 `-x` 启用已执行行的调试输出，可让其打印出脚本执行过程中的所有语句或脚本内调试部分脚本

**示例1**：跟踪脚本的执行

```bash
$ cat example_script.sh
#!/bin/bash
echo "Hello $USER,"
echo "Today is $(date +'%Y-%m-%d')"

$ bash -x example_script.sh 
+ echo 'Hello root,'
Hello root,
++ date +%Y-%m-%d
+ echo 'Today is 2022-09-02'
Today is 2022-09-02
```

**示例2**：调试部份的脚本

```bash
#!/bin/bash
set -x   # Enable debugging
# some code here
set +x   # Disable debugging output.

#!/bin/bash
echo "Hello $USER,"
set -x
echo "Today is $(date %Y-%m-%d)"
set +x

$ bash -x example_script.sh 
./example_script.sh 
Hello root,
++ date %Y-%m-%d
date: invalid date ‘%Y-%m-%d’
+ echo 'Today is '
Today is 
+ set +x
```

