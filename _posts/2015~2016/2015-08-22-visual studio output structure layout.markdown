---
layout: post
title: 如何用VisualStudio查看结构体布局
categories: Windows
description: 如何用VisualStudio查看结构体布局
keywords: 
---

&emsp;&emsp;假设在每个系统的structA  结构不同，我们在windbg看了以后直接拿来用，自己定义成结构体，如何来验证这个结构体内存布局是否和windbg一致。当然笨办法是自己一个个成员数过去，然而人眼总有看错的时候，你承认吧~~。这里用一个极其巧妙的方式解决这个问题：  
&emsp;&emsp;在vs当前工程中，添加了结构体定义，并编译成功后，解决方案视图，工程 右键 -> 属性 -> C/C++ -> 命令行 -> 其它选项  
* 输出单个结构体结构：`/d1reportSingleClassLayoutstructA`，注意structA可以是任何类名、结构体名、联合体 等结构型结构，注意structA之前并没有空格。这是输出某个结构体内存布局的方式，如果要
* 输出所有工程引用到的结构体布局：`/d1reportAllClassLayout`，结果会非常庞杂

