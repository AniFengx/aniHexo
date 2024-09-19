---
title: Hexo使用指北
date: 2022-10-16 21:23:37
tags: [Hexo]
categories: [Hexo]
---

第一次使用Hexo还是在2017年，那时候到处了解学习新兴技术，go也是在那段时间听说并进行了简单学习；当时应该是在搜如何建一个个人博客，然后就搜到了Hexo的相关文章，因为其借用GitPage相关特性无需自建服务器，直接依托github就能建立个人博客，并且也能支持搜索引擎收录，这些亮点在当时的我看来还是很酷的（现在依然觉得很酷~），但是当时在研究完如何安装并成功放到GitPage之后，因为没有良好的书写文档的习惯，个人博客也就搁置了，不过当时还是很惊叹于Hexo运行后的博客页面，风格很吸引人。

最近使用github时，看了看自己的仓库发现了这个被遗弃的博客仓库，正好最近意识到了文档的重要性，并且也在尝试书写一些，而且用的也是markdown格式，能够无缝衔接到Hexo上，就准备把这个仓库重新用起来，希望不算太晚吧。

{% note info %} 说起来，markdown也是2017那年学习使用的，因为当时正值毕业季，需要写个人简历，可能大部分人都是用的智联吧，后来我同学推荐我用乔布简历，写完效果拔群\~比智联不知道高到哪里去了，后来同学又推荐我使用markdown，简单的编写规则加上自由的样式控制可以写出非常好看的简历，并且也足够小众足够酷（但是现在基本程序员已经都会用这个工具了），这才是程序员写个人简历的工具\~从此之后我写文档基本都是用markdown来写 {% endnote %}

<!-- more -->

## Hexo

### Hexo安装方式

鉴于国情，很多人学习技术更多的是在csdn或者其他个人博客，通过别人学习后总结的文章来学习，确实这样学起来更快，并且很多官方网站都是英文的，而csdn或者个人博客相当于一个汉化的过程，但是csdn会存在一个问题就是文章中的内容会出现过时的情况，所以还是建议直接前往官网学习，并且[Hexo官网](https://hexo.io/zh-cn/docs/ "Hexo官网")是有简体中文的。

#### 安装git

直接去git官网下载64位的exe安装文件即可

#### 安装node

node版本是个坑，很多npm包是对node版本有要求的，并且有的时候过新的node版本也会导致npm包失败，我尝试Hexo安装成功的node环境版本是*14.16.1*

#### 安装Hexo

npm全局安装Hexo

``` shell
$ npm install -g hexo-cli
```

Hexo初始化

``` shell
$ hexo init
$ npm install
```

配置文件

_config.yml是Hexo的配置文件，可以在其中设置一些基本参数，例如博客名称、作者名称及一些插件的配置信息

#### 安装主题NEXT

Hexo默认主题外观还是比较一般的，Hexo支持很多扩展主题，我使用的是NexT主题，国内大部分网站也是推荐使用这个

但是这就引出了一个坑，因国内大部分介绍Hexo的博客文章都停留在2020年或2020年以前了，假如没有去Hexo官方学习安装方式，而是直接按照国内博客来，那么你安装主题的命令一定是

``` shell
$ git clone https://github.com/theme-next/hexo-theme-next themes/next
```

在安装完这个主题后，你会发现使用起来都很正常，但是几个第三方插件却会出现不好使的情况，并且也没有任何参考解决方案，例如当你要用leancloud来当浏览量插件时，页面会出现报错的情况。这实际上是因为NexT原作者出现了长期不在线的情况，导致其他成员无法管理已有仓库，所以重新创建了新的分支[解释博客](https://github.com/next-theme/hexo-theme-next/issues/4 "解释博客")

正确的安装主题命令

``` shell
$ git clone https://github.com/next-theme/hexo-theme-next themes/next
```

> 命令后缀的themes/next不必须按照示例来，next是指clone后的所在文件夹名，这个可以任意命名，只需要clone后，将根目录下的_config.yml中theme参数配置为对应的文件夹名就可以了

Hexo新版本下，修改主题的配置文件有了新的方式，只需要在根目录下建立_config.xxx.yml，xxx对应的是themes下使用主题所在的文件夹名

### Hexo常用命令

- `hexo clean` 删除编译文件
- `hexo g` 编译文件
- `hexo d` 推送编译文件到远端。如果存储在 github 上，需配置 setting 中的 **SSH keys**

### 常用 Markdown 语法

#### Note 引用

Note 引用包含 `success / primary / warning / default / info / danger` 类型

``` Markdown
{% note warning %} warning {% endnote %}
````

