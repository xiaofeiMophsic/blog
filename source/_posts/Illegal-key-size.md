---
title: Illegal key size
author: 小飞
date: 2017-02-16 18:21:30
tags:
---
## 问题描述
在Android studio中使用 Robolectric 对Android进行单元测试的时候发现，在执行到AES加密算法的时候抛出`java.security.InvalidKeyException: Illegal key size`。查找资料后才发现美国对出口的密钥长度作了限制，需要下载下面两个jar包。    
地址：http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html

下载完成之后放到`$android-studio-home$/jre/jre/lib/security`目录下覆盖原文件即可。
