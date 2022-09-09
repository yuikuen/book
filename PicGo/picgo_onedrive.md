# PicGo Onedrive

> **前言**：之前使用 Github 作为 Hexo 博客，而 Gitee 作为博客图床使用，但 22/03/27 之后，Gitee 图床废了！！具说是官方添加了防盗链导致，虽然现在 Typora + PicGo + Gitee 的图床是可以正常显示了，我的笔记都得以查看，但我博客还是无法显示图片，而 Gitee 作为图床，体质也不算太好(有 1M 图片大小的限制)，为了安全起见，还是建议更换图床。

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/3eddd173846db4ea4004f8034cce489e.png)

## 需求分析

- 不想额外花费，可免费决不采用收费方式，比如使用 OneDrive、GoogleDrive
- 能帮助我管理图片，作为免费的图床，Typora & Blog 都能在网络获取图片

经过多番查证，为了方便自己的习惯，具体方案还是选择 PicGo +  OneDrive(Amazon S3) + Typora + Blog。PicGo 不提供存储，但它有个 Amazon S3 插件，配合 PoweredBy.Cloud + S3 插件，可实现 OneDrive 图床服务

1）为什么使用 OneDrive

Amazon S3 是一个产品，主流的存储产品，比如阿里云 OSS、华为 OSS、腾讯 COS 等，虽然存储不贵，但还是有带宽、流量等费用问题；

而 OneDrive 免费 5G 空间，容量不足还能升级，将来文件迁移也方便，也自带杀毒和备份恢复功能（GoogleDrive 免费 15G 空间，但国内上传和访问都是问题，也就放弃了）

2）PicGo 的好处

[PicGo](https://github.com/Molunerfinn/PicGo) 是开源产品，可以在 Github 上找到，在 Win、Linux、MacOS 等不同平台都提供了程序，并且还可以安装不同的插件来配合各大存储服务；

## PicGo 安装

安装方式都较为简单，在此就不再详说，可参照 PicGo Github

1）**安装后打开主界面**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/da8c34e3a24806ab88854faae58451a9.png)

2）**选择插件设置-搜索 S3 -选择安装 S3 1.1.4(picgo amazon s3 uploader)**

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/a6bdcaf760592f201da335997203183f.png)

> 注意：必须先安装 [node.js](https://nodejs.org/en/) 才能安装插件，请自行提前安装一下

## PoweredBy.Cloud 配置

> [PoweredBy.Cloud](https://poweredby.cloud/) 是一个将 GoogleDrive 和 OneDrive 变成图床的工具，存在网盘里文件的支链我们是无法获取的，如果能获取到文件的支链，那就能将 GoogleDrive 和 OneDrive 当作图床用，它就是这样一个免费工具，提供网盘的直链，而且使用起来没有任何限制

1）注册账号 

1. 打开 [PoweredBy.Cloud](https://poweredby.cloud/) 并注册，注册非常简单，只需要提供一个可收邮件的邮箱，登录地址会发送到邮箱
2. 在邮箱里找到登录地址，并点击登录地址登录到 PoweredBy.Cloud
3. 在 PoweredBy.Cloud 的控制台里添加一个 site，例如：http://demo.stdcdn.com  选择 GoogleDrive 或者 OneDrive 作为存储空间(PoweredBy.Cloud 会在你的网盘里创建一个 [http://demo.stdcdn.com](http://demo.stdcdn.com/) 的文件夹)

注册site截图如下，Site Name 将在后面使用

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/f8a423e911cf882ce7b45bc9c82096e8.png)

2）创建 Developer 密钥(S3)

点击 Developer 选项->点击 Create Access Key 按钮->输入任意 key 名称(仅做备注)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/fda4209f4bd3370934b82f5861cdbb94.png)

## PicGo 配置 S3

配置 S3 的关键参数在创建密钥后，access_key、secret_access_key 都已经出现，剩下的还有两个关键值

```js
//下面是两个关键参数
//固定值,endpoint 自定义节点
endpoint_url='https://stdcdn.com'
//桶,bucketName 这就是你的SiteName。
bucket="yuikuen"
//自定义节点
https://stdcdn.com
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/1ec466a959fe032877e4fce8c0bc987b.png)

**测试图片上传效果**

1）使用 PicGo 测试上传效果，最终会在我们的 OneDrive 中出现

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/ccd5adfe012727d0212a7a766d204878.png)

2）Typora 中，可以在图像中配置上传图片，并且上传可以进行验证图片上传选项操作

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/a433a9e8a646062b9b12ab7065d8b69f.png)