---
title: chrome浏览器内置谷歌翻译无法使用解决方案
date: 2022-10-10 16:21:31
tags: [chrome]
categories: [开发工具]
comments: true
---

google近期关闭了国内谷歌翻译网站，但是chrome浏览器内置的翻译功能用起来比较顺手，可以通过修改host的方式临时解决下

### 修改host教程

#### 进入etc文件夹

+ mac: 右键访达 - 前往文件夹 - 输入 "etc/hosts"
+ windows: C:\Windows\System32\drivers\etc

修改文件,添加以下内容：

```
203.208.40.66 translate.google.com
203.208.40.66 translate.googleapis.com
```

保存修改后，chrome即可正常使用