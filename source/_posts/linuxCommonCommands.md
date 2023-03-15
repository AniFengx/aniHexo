---
title: linux常用命令
date: 2023-03-15 16:55:14
tags: [linux]
categories: [linux]
---

平常查询日志等工作都是通过ELK来完成，并且ELK用起来也比较方便，但是有时候存在单条日志信息过多导致ELK收集器忽略或者截断的情况，导致无法查看需要的全部日志来定位问题，所以会需要连到服务器上去查看日志，这时候就需要用到一些linux命令，在此记下方便以后直接复制使用

### 查看日志的命令

#### tail

服务器启动失败，只需要查看最近的失败日志；查看当前服务运行情况等，可以使用本命令

`-f`实时读取最新内容，默认10行
``` shell
tail -f task_info.log
```

`-n`指定要读取的行数，读取最后n行
``` shell
tail -n 30 task_info.log
```

<!-- more -->

#### cat

查看文件内容，服务器上大文件使用时要注意控制显示内容

``` shell
cat task_info.log
```

#### grep

根据一些关键词搜索要查看的内容，一般结合cat使用

``` shell
cat task_info-20221216.5.log | grep '锁座'
```

> 一些额外的选项，方便查看日志
- -A n : 如果匹配成功，则将匹配行及其后n行一起打印出来<br/>
- -B n : 如果匹配成功，则将匹配行及其前n行一起打印出来<br/>
- -C n :如果匹配成功，则将匹配行及其(前后)n行一起打印出来<br/>
- --color=auto ：可以将找到的关键词部分加上颜色的显示

***e.g.***

``` shell
cat task_info-20221216.5.log | grep -n -A2 -B1 --color=auto '锁座'
```

#### sz

从服务器上下载文件

``` shell
sz apiclient_cert.p12
```