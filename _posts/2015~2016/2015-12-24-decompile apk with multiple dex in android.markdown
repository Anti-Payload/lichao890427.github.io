---
layout: post
title: 如何反编译多dex的apk
categories: Reverse_Engineering
description: 如何反编译多dex的apk
keywords: 
---

&emsp;&emsp;今天遇到了一个apk中存在(将资源分成)多个hex的情况，这种文件会影响jeb apkide的处理和跳转，因此有必要将多个dex合并。apk本身是压缩文件，类似zip，可以用winrar打开，而dex文件则类似于压缩文件  
* 首先考虑合并hex，发现不存在这样的工具
* 其次考虑反编译hex得到jar，然后合并jar，然后编译成dex   此种方法，由于dex2jar工具有bug而导致有些文件缺失
* 最后我考虑dex2smali，将dex解压到同一目录，然后smali2dex合并成dex，最后用aapt add替换apk包中原始dex，过程如下：  

工具：dex2jar工具集、aapt、winrar

* 1.用winrar将demo.apk中多个dex解压  classes.dex  classes2.dex  
* 2.解压dex到同一目录，运行  
`d2j-dex2smali.bat classes.dex --force -o tmp`  
`d2j-dex2smali.bat classes2.dex --force -o tmp`  
* 3.合并为同一dex  
`d2j-smali.bat -o classes.dex tmp`  
* 4.用winrar删除apk的旧dex文件，将dex加入apk中  
`aapt add demo.apk classes.dex`  

