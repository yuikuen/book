# Gerrit Plug

> Gerrit 可以通过不同的插件实现各种功能，这跟 Jenkins 差不多

1）通过 [GerritForge](https://gerrit-ci.gerritforge.com/)，可以找到自己想要的插件

注：插件请根据版本自身安装的版本进行选择

![20220714150219.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714150219.png)

2）以 reviewers 插件为例，可以在搜索框输入关键字进行检索

![20220714150450.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714150450.png)

3）获得对应的 [Jar包下载地址](https://gerrit-ci.gerritforge.com/job/plugin-reviewers-bazel-stable-3.5/lastSuccessfulBuild/artifact/bazel-bin/plugins/reviewers/reviewers.jar)，下载插件并放置在 `$GERRIT_SITE/plugins` 目录下，然后重启服务，会自动加载此目录下的插件

```bash
$ cd /usr/local/gerrit/review_site/plugins
$ sudo wget https://gerrit-ci.gerritforge.com/job/plugin-reviewers-bazel-stable-3.5/lastSuccessfulBuild/artifact/bazel-bin/plugins/reviewers/reviewers.jar
--2022-07-14 15:10:03--  https://gerrit-ci.gerritforge.com/job/plugin-reviewers-bazel-stable-3.5/lastSuccessfulBuild/artifact/bazel-bin/plugins/reviewers/reviewers.jar
Resolving gerrit-ci.gerritforge.com (gerrit-ci.gerritforge.com)... 8.26.94.23
Connecting to gerrit-ci.gerritforge.com (gerrit-ci.gerritforge.com)|8.26.94.23|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 46865 (46K) [application/java-archive]
Saving to: ‘reviewers.jar’

100%[=========================================================================================================>] 46,865      10.7KB/s   in 4.3s   

2022-07-14 15:10:12 (10.7 KB/s) - ‘reviewers.jar’ saved [46865/46865]

$ sudo chmod 755 reviewers.jar && sudo chown gerrit. reviewers.jar
$ sudo ./gerrit/review_site/bin/gerrit.sh restart
Stopping Gerrit Code Review: OK
Starting Gerrit Code Review: OK
```

![20220714152813.png](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/20220714152813.png)

查看插件是否安装成功
