---
layout: post
title: 解决Android Studio Gradle下载慢
categories: Android
description: 解决Android Studio Gradle下载慢
keywords: 
---

# 解决Android Studio Gradle下载慢

## 原因
&emsp;&emsp;每次新建工程，Android都会下载特定版本gradle到本地，而由于下载逻辑的拙劣，经常出现一直卡死的情况。用everything工具查找`gradle .part`，可以找到当前在下载的gradle是2个文件：

```Txt
gradle-*-all.zip.part
gradle-*-all.zip.lck
```

&emsp;&emsp;此时我们杀死AndroidStudio进程删除它们，并从`http://services.gradle.org/distributions/`下载合适的文件，下载到该目录，重启AndroidStudio会发现已经将新文件解压，很快就完成了gradle初始化过程！！！  
&emsp;&emsp;不明白AS为什么不提升一下用户体验，自带gradle下载实在太慢
