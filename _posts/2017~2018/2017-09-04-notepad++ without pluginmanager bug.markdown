---
layout: post
title: 解决Notepad++没有PluginManager的Bug
categories: Miscellaneous
description: 解决Notepad++没有PluginManager的Bug
keywords: 
---

# 解决Notepad++没有PluginManager的Bug

## 问题描述

&emsp;&emsp;今天在使用notepad++时想安装markdown插件，却发现没有插件菜单下没有plugin manager，总结原因可能有如下：

1.官方不为64位npp提供plugin manager

2.npp7.5版本及以后的文件中没有PluginManager.dll因此没有plugin manager

&emsp;&emsp;而我自己的npp确实是最新7.5.1版，综上

## 解决如下：

1.安装x86版本notepad++

2.从7.4.2版本zip中取出pluginmanger.dll放到你的npp的plugin目录下即可
