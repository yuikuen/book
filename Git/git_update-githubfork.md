# Git Update Github-Fork

> 使用 Git 更新 Github 上 Fork 的项目

1）首先 Clone 项目到本地

```bash
$ git clone git@github.com:yuikuen/k8s.git
Cloning into 'k8s'...
remote: Enumerating objects: 1091, done.
remote: Counting objects: 100% (459/459), done.
remote: Compressing objects: 100% (296/296), done.
Receiving objectremote: Total 1091 (delta 176), reused 409 (delta 145), pack-reused 632
Receiving objects: 100% (1091/1091), 5.24 MiB | 1.27 MiB/s, done.
Resolving deltas: 100% (452/452), done.
```

2）查看当前远程版本库地址

```bash
$ git remote -v
origin  git@github.com:yuikuen/k8s.git (fetch)
origin  git@github.com:yuikuen/k8s.git (push)
```

3）添加原项目 git 地址到本地版本库

```bash
$ git remote add upstream https://github.com/dotbalo/k8s.git
```

4）检查版本是否添加成功

```bash
$ git remote -v
origin  git@github.com:yuikuen/k8s.git (fetch)
origin  git@github.com:yuikuen/k8s.git (push)
upstream        https://github.com/dotbalo/k8s.git (fetch)
upstream        https://github.com/dotbalo/k8s.git (push)
```

5）原项目更新内容同步到本地

```bash
$ git fetch upstream
remote: Enumerating objects: 25, done.
remote: Counting objects: 100% (25/25), done.
remote: Compressing objects: 100% (16/16), done.
remote: Total 21 (delta 9), reused 14 (delta 4), pack-reused 0
Unpacking objects: 100% (21/21), 51.27 KiB | 905.00 KiB/s, done.
From https://github.com/dotbalo/k8s
 * [new branch]      add-license-1 -> upstream/add-license-1
 * [new branch]      master        -> upstream/master
```

6）查看本地分支

```bash
$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/add-license-1
  remotes/origin/master
  remotes/upstream/add-license-1
  remotes/upstream/master
```

7）同步更新内容到本地对应分支

```bash
$ git merge upstream/master
Already up to date.
```

8）提交更新内容到 fork 地址

```bash
$ git push
Enumerating objects: 1, done.
Counting objects: 100% (1/1), done.
Writing objects: 100% (1/1), 241 bytes | 241.00 KiB/s, done.
Total 1 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:yuikuen/k8s.git
   2896ca2..00d23be  master -> master
```

最后自行到 Github 仓库查看是否有 update 成功；