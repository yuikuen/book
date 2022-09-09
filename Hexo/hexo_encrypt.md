# Hexo Encrypt

> Hexo 博客配置 [hexo-blog-encrypt](https://github.com/D0n9X1n/hexo-blog-encrypt) 插件实现文章加密

## 安装及配置

```bash
$ npm install --save hexo-blog-encrypt
# 修改_config.yaml
# Security
##
encrypt:
    enable: true
    password: 123456 # 默认密码
```

通过修改 `_config.yaml` 开启并设置`默认密码`，若文章加密并未声明独立密码即可通过默认密码解锁

- **快速使用**

将 "password" 字段添加到文章信息头即可

```markdown
---
title: Hello World
date: 2016-03-30 21:18:02
password: hello
---
```

## 高级设置

- 文章信息头加密

```markdown
---
title: Hello World
tags:
- encryptAsDiary
date: 2016-03-30 21:12:21
password: mikemessi
abstract: Here's something encrypted, password is required to continue reading.
message: Hey, password is required here.
wrong_pass_message: Oh, this is an invalid password. Check and try again, please.
wrong_hash_message: Oh, these decrypted content cannot be verified, but you can still have a look.
---
```

- 全站默认设置

```bash
# Security
encrypt: # hexo-blog-encrypt
  abstract: Here's something encrypted, password is required to continue reading.
  message: Hey, password is required here.
  tags:
  - {name: encryptAsDiary, password: passwordA}
  - {name: encryptAsTips, password: passwordB}
  wrong_pass_message: Oh, this is an invalid password. Check and try again, please.
  wrong_hash_message: Oh, these decrypted content cannot be verified, but you can still have a look.
```

- 禁用加密

```bash
---
title: Callback Test
date: 2019-12-21 11:54:07
tags:
    - A Tag should be encrypted
password: ""
---

Use a "" to disable tag encryption.
```

**配置优先级**

文章信息头 > `_config.yml` (站点根目录下的) > 默认配置
