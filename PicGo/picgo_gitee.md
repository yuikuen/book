# PicGo Gitee

> **前言**：一开始都是使用云笔记的，但不便于保存或查看，后面转到了在线博客。而在线博客的编辑器坑太多了，所以开始尝试不同的方案，现暂敲定采用 "Typora + GitHub + Hexo" 方式，而图床一开始也是保存本地的，但在上传时又过于慢，因此改成 "Typora + GitHub + Hexo + PicGo(Gitee)" 来实现 Markdown 图床。

## 程序安装

- PicGo (Windows、Manjaro Linux)
- PicGo-plugin-gitee-uploader 插件

Windows 与 Manjaro Linux 的安装方式都较为简单，在此就不再详说，可参照 PicGo Github

**安装后打开主界面**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207154537047.png)

**选择插件设置-搜索 gitee -选择安装 gitee-uploader 1.1.2**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207154656271.png)

> 注意：必须先安装 [node.js](https://nodejs.org/en/) 才能安装插件，请自行提前安装一下

## 建立 Gitee 图床库

注册码云方法略过，直接创建仓库即可，新建仓库要点如下：

- 仓库名称不可重复
- 仓库设为公开，便于博客访问

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207155208280.png)

## 配置 PicGo

打开软件，选择图床设置 gitee

- repo：用户名/仓库名称
- branch：分支，默认为 master
- token：码云的私人令牌
- path：路径，根据实际填写
- customPath：提交消息，这一项和下一项customURL都不用填。在提交到码云后，会显示提交消息，插件默认提交的是 `Upload 图片名 by picGo - 时间`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207160229270.png)

**Token 获取，在个人设置创建就有了**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207160827190.png)

根据实际需求进行勾选，默认即可；

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207161003426.png)

需要验证一下密码，验证密码之后会出来一串数字，这一串数字就是你的 token，注意：这个令牌只会明文显示一次，建议在配置插件的时候再来生成令牌，直接复制进去

## 配置 Typora

文件--偏好设置--图像，根据自身环境配置，点击验证图片上传选项即可，下以 *Windows* 为例

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207161531543.png)

