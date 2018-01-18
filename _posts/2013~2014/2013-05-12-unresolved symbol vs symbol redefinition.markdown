---
layout: post
title: Unresolve symbol与Redefinition symbol
categories: C/C++
description: C/C++常见编译错误
keywords: 
---

# C/C++常见编译问题：未找到符号和符号重定义
&emsp;&emsp;使用Gcc/Vs编译器，初学者(不限于)经常犯的错误包括这两个问题。
* 对于未找到符号(Unresolve symbol)，就是编译器无法在头文件.h，源文件.c/.cpp和静态库.lib/.a中找到相关的函数实现。所以极有可能的原因是编译时未链接该函数，解决办法如下：
* * 先文档(CSDN)，后手工方式搜索函数/符号定义的具体位置，注意定义和声明的区别。
* * 搜到后若定义存在.h文件中，则只需要包含到公共头文件即可
* * 若定义存在.c/.cpp文件中，则要将该文件添加到工程，并添加到编译设置中
* * 若定义在.lib/.a文件中，则需要将文件添加到链接选项中
* * 对于前两种情况比较好判断，对于存在.lib/.a的情况，通过dumpbin -export类似命令定位是否存在

* 对于符号重定义(Redefinition)，就是编译器在.h/.c/.cpp/.lib/.a找到了2个以上的定义存在，这时候需要排查以下几点
* * 是否真的已经存在官方实现，自己又造轮子了
* * 是否将变量/符号定义写到.h文件中，而.h又被多个.c/.cpp引用，若是这样，只需将.h中的定义改成声明，可以根据c/c++用extern/extern "C" 声明，再将定义随意写到一个.c/.cpp中即可，比如将1.h中的int i = 0;改成extern int i;然后再1.c加上int i=0;
