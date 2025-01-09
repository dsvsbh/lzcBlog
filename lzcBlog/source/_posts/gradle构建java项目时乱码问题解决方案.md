---
title: gradle构建java项目时乱码问题解决方案
date: 2024-10-11 22:25:16
tags: gradle
categories: 业务场景 
---

# gradle构建java项目时乱码问题解决方案

今天在公司接到了一个需求，项目是用gradle构建的，因为之前从来没用过gradle，第一次用发现在构建的时候一直输出乱码,并且构建失败,最后找到了解决方案。

如果项目是用UTF-8编码编写的那么应该使用UTF-8编码构建项目。

解决方法如下：

1. idea点击帮助=>编辑自定义虚拟机选项

2. 添加代码`-Dfile.encoding=UTF-8`

3. 重启idea

重新构建gradle项目观察日志输出没有乱码即可
