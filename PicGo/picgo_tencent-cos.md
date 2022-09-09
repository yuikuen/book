# PicGo Tencent-Cos

> **前言**：之前搭建了各种图床方法，为的就是日常笔记和博客使用。从本地转到云上服务，尝试了各种白嫖方法，体验还是各种不适。不是慢就是打不开，还有写文章时上传失败的问题，遂决定更换成付费方案；
>
> 查阅了各种云上服务商的方案，本有折腾的心搞个 VPS 做图床服务器，但最近没优惠，暂搁浅下来，综合考虑还是选择了腾讯云 COS + PicGo 来部署图床。

## 配置 COS

1）购买腾讯云 COS

注册账号与购买过程，请参考 [腾讯云对象存储 COS](https://cloud.tencent.com/product/cos) 介绍，根了解之前是 50G 永久免费额度，现调整成 50G 免费六个月，足于日常使用，其次续约也不是很贵。[新用户还有个专享礼包可以体验 1元 50G/1年的抢购活动](https://cloud.tencent.com/act/pro/cos#%E6%96%B0%E7%94%A8%E6%88%B7%E4%B8%93%E4%BA%AB%E7%A4%BC%E5%8C%85)

2）创建存储桶

进入 [控制台](https://console.cloud.tencent.com/cos) →【对象存储】→【存储桶列表】→【创建存储桶】

![image-20220518173916057](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220518173916057.png)

访问权限选择**公有读私有写**，否则图片无法读取，其他的根据自己往下填写就可以

3）密钥配置

点击【密钥管理】→【云API密钥】→【新建密钥】

![image-20220518174403816](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220518174403816.png)

## 配置 PicGo

1）打开软件，点击【腾讯云COS】，首先将我们上面创建的 API 密钥粘贴

![image-20220518174752336](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/images/image-20220518174752336.png)

- COS 版本：V5 (现版本基本为 V5，所以勾选 V5 版本)
- Secretld、SecretKey、APPID：云 API 密匙新建的，如旧的也可直接使用
- 存储空间名：Bucket 名字，存储桶名称
- 存储区域：存储桶所属地域
- 存储路径：桶【配置管理】的【文件列表】中创建，可自定义
- 自定义域名：空即可

腾讯云 COS 的 `json` 配置如下，仅供参考

```json
{
  "secretId": "",
  "secretKey": "",
  "bucket": "",            // 存储桶名，v4和v5版本不一样
  "appId": "",
  "area": "",              // 存储区域，例如ap-beijing-1
  "path": "",              // 自定义存储路径，比如img/
  "customUrl": "",         // 自定义域名，注意要加http://或者https://
  "version": "v5" | "v4"   // COS版本，v4或者v5
}
```