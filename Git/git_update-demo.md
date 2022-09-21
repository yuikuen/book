# Git Update-Demo

> Git 简单操作上传代码至远程仓库

本地创建了一个 Demo 工程项目，现通过 **命令行** 将该项目上传到 github 或者 gitlab 远程仓库，具体操作流程如下：

1）建立本地 git 仓库，cd 到你的本地项目根目录下，执行 git init 命令

```bash
$ cd demo 
$ git init
# 进入本地工程根目录，init后将此目录变成git可管理的仓库
```

2）将本地项目工作区的所有文件添加到暂存区

```bash
$ git add .
# 小数点 “.” ，意为添加文件夹下的所有文件，可换成具体文件名
```

3）将暂存区的文件提交到本地仓库

```bash
$ git commit -m "注释说明"
```

4）在 github 或者 gitlab 上创建新的 repository

略过

5）将本地代码仓库关联到 github/gitlab 上

```bash
$ git remote add origin git@yk-gitlab.net:root/intell-project.git
```

错误 `fatal:remote origin already exists` 解决方案

```bash
$ git remote rm origin
$ git remote add origin git@yk-gitlab.net:root/intell-project.git
# 删除仓库日志再重新关联
```

6）将代码由本地仓库上传到 github 远程仓库，依次执行下列语句

- 获取远程库与本地同步合并（如果远程库不为空必须做这一步，否则后面的提交会失败）：

```bash
$ git pull --rebase origin master  
# 不加这句可能报错，原因是 github 中的 README.md 文件不在本地仓库中,可以通过该命令进行代码合并
```

- 把当前分支 master 推送到远程，执行此命令后有可能会让输入用户名、密码：

```bash
$ git push -u origin master  
# 执行完之后如果无错误就上传成功了，需要提示的是这里的 master 是 github 默认的分支，
# 如果你本地的当前分支不是 master，就用git checkout master命令切换到master分支，
# 如果你想用本地当前分支上传代码，则把第6步的命令里的 master 切换成你的当前分支名即可。
```
