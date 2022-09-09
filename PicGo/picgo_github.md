# PicGo Github

> **前言**：Gitee 作为图床的功能无法使用，虽然 Typora 还是正常显示图片，但 Blog 全 404 挂掉了。中间试过换成 Onedrive 作图床，白嫖 5G 免费空间加上 CDN，速度还是可能的，也方便管理(自带误删恢复功能)。但不久后发觉微软的大部分程序都给河蟹了…如：Onedrive、Hotmail 等都无法访问(其实可以嘀，只是要梯子而已！！)，最终还是先改成 Github + cdn.jsdelivr 应付着…

## 程序安装

- 安装方式都较为简单，在此就不再详说，可参照 PicGo Github。直接下载程序安装，**安装后打开主界面**

## 创建图库

- 登录 Github 后，创建一个新的仓库，一般只需要个仓库名即可，其它默认； (注意权限选择公开 Public )

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220409154656553.png)

- 创建一个 `token`，点击头像处 `Settings -> Developer settings -> Personal access tokens`，最后点击 `generate new token`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220409155120000.png)

- 填写用途(任意即可)，勾选 `repo`

​       `token` 生成，注意它只会显示一次，所以最好把它复制下来存好，方便下次使用，否则下次有需要重新新建

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220409155243239.png)

## 配置 PicGo

配置 PicGo，依次打开 图床设置 -> Github 图床

- 仓库名：yuikuen/picgo-cdn_images 刚创建的仓库
- 分支名：main 现一般都为这个
- Token：****** 刚生成的 access token
- 路径：img/ 仓库下创建的文件目录
- 域名：https://cdn.jsdelivr.net/gh/用户名/仓库名   [jsDelivr](https://www.jsdelivr.com/) 免费加速

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220409155637537.png)

##  配置 Typora

文件--偏好设置--图像，根据自身环境配置，点击验证图片上传选项即可，下以 *Windows* 为例

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20211207161531543.png)