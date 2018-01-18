---
layout: post
title: 变速齿轮原理浅析
categories: Reverse_Engineering
description: 变速齿轮原理浅析
keywords: 
---

# 变速齿轮原理浅析

&emsp;&emsp;变速齿轮包括3个部分：
* GearNT.exe为主界面，处理界面响应，增删程序、快捷键等
* GearNtKe.dll用于修改api以控制速度，为核心部分
* Hook.dll主要做了键盘hook和消息hook，配合GearNT.exe

&emsp;&emsp;选中进程加速后，GearNtKe.dll调用AddProcess，所做的工作是将自身注入到进程中，替换如下api：

|函数                   |行为                                |
|-----------------------|------------------------------------|
|CreateProcessW         |再次注入以在新进程中加载GearNtKe.dll| 
|CreateProcessA         |再次注入以在新进程中加载GearNtKe.dll| 
|GetMessageTime         |修改返回的time                      |
|GetTickCount           |修改返回的count                     |
|QueryPerformanceCounter|修改PerformanceCount参数            |
|SetTimer               |修改时间间隔                        |
|timeGetTime            |修改得到的time                      |
|timeSetEvent           |修改delay                           |

&emsp;&emsp;都是与时间有关的api，他们被进行了API hook，换成了新函数，而这些新函数都按情况分别进行了处理，尤其是传入参数，之后再调用原始api，理解了上述过程，我不禁对作者创造的这个工具赞叹，想法太妙了，也达到了效果。该软件早就不更新了，我这Win7 x64不好使了，等我弄出了源码，会改造成通用一些的工具。

