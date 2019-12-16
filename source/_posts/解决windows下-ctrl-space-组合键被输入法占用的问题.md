---
title: 解决windows下 ctrl + space 组合键被输入法占用的问题
author: 小飞
date: 2017-01-12 15:36:54
tags: [其他]
---

每次在intellij下想用通过`ctrl+space`自动完成功能的时候，总是被输入法切换给占用，好难受，之好自己动手改注册表了。
办法很简单，打开注册表：如下图           
![注册表](/images/hot_key.png)
只需要将`key modifier` 和`virtual key` 对应的值修改成图中对应的值，重启电脑就好了。
