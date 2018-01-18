---
layout: wiki
title: Android逆向分析笔记
categories: Reverse_Engineering
description: Android逆向分析笔记
keywords: 
---

<!-- TOC -->

- [概述](#%E6%A6%82%E8%BF%B0)
    - [分析步骤](#%E5%88%86%E6%9E%90%E6%AD%A5%E9%AA%A4)
        - [通用逆向分析步骤](#%E9%80%9A%E7%94%A8%E9%80%86%E5%90%91%E5%88%86%E6%9E%90%E6%AD%A5%E9%AA%A4)
        - [安卓上APK调试步骤：](#%E5%AE%89%E5%8D%93%E4%B8%8Aapk%E8%B0%83%E8%AF%95%E6%AD%A5%E9%AA%A4%EF%BC%9A)
        - [安卓上linux程序调试步骤：](#%E5%AE%89%E5%8D%93%E4%B8%8Alinux%E7%A8%8B%E5%BA%8F%E8%B0%83%E8%AF%95%E6%AD%A5%E9%AA%A4%EF%BC%9A)
    - [分析工具](#%E5%88%86%E6%9E%90%E5%B7%A5%E5%85%B7)
        - [APK改之理](#apk%E6%94%B9%E4%B9%8B%E7%90%86)
        - [JD-GUI](#jd-gui)
        - [JEB](#jeb)
        - [Dex2jar工具集](#dex2jar%E5%B7%A5%E5%85%B7%E9%9B%86)
        - [Apktool反编译&打包工具](#apktool%E5%8F%8D%E7%BC%96%E8%AF%91%E6%89%93%E5%8C%85%E5%B7%A5%E5%85%B7)
    - [常见文件格式](#%E5%B8%B8%E8%A7%81%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F)
        - [Apk](#apk)
            - [使用aapt解析xml](#%E4%BD%BF%E7%94%A8aapt%E8%A7%A3%E6%9E%90xml)
        - [Dex](#dex)
        - [Jar](#jar)
        - [Odex](#odex)
        - [Aar](#aar)
        - [So](#so)
        - [工具转换图](#%E5%B7%A5%E5%85%B7%E8%BD%AC%E6%8D%A2%E5%9B%BE)
        - [Android设备上重要目录](#android%E8%AE%BE%E5%A4%87%E4%B8%8A%E9%87%8D%E8%A6%81%E7%9B%AE%E5%BD%95)
- [Java层](#java%E5%B1%82)
    - [常用工具](#%E5%B8%B8%E7%94%A8%E5%B7%A5%E5%85%B7)
        - [adb](#adb)
        - [aapt](#aapt)
        - [am & pm](#am-pm)
    - [有源码调试APK](#%E6%9C%89%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95apk)
        - [Android studio](#android-studio)
        - [Adb wifi](#adb-wifi)
    - [无源码调试apk](#%E6%97%A0%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95apk)
        - [使用AndroidStudio和Apktool工具调试](#%E4%BD%BF%E7%94%A8androidstudio%E5%92%8Capktool%E5%B7%A5%E5%85%B7%E8%B0%83%E8%AF%95)
        - [使用jdb调试](#%E4%BD%BF%E7%94%A8jdb%E8%B0%83%E8%AF%95)
            - [查看参数](#%E6%9F%A5%E7%9C%8B%E5%8F%82%E6%95%B0)
            - [设置源码从而进行逐行调试](#%E8%AE%BE%E7%BD%AE%E6%BA%90%E7%A0%81%E4%BB%8E%E8%80%8C%E8%BF%9B%E8%A1%8C%E9%80%90%E8%A1%8C%E8%B0%83%E8%AF%95)
            - [行断点：](#%E8%A1%8C%E6%96%AD%E7%82%B9%EF%BC%9A)
            - [初始断点](#%E5%88%9D%E5%A7%8B%E6%96%AD%E7%82%B9)
            - [调试命令](#%E8%B0%83%E8%AF%95%E5%91%BD%E4%BB%A4)
    - [无源码调试dex](#%E6%97%A0%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95dex)
- [Linux层](#linux%E5%B1%82)
    - [常用工具](#%E5%B8%B8%E7%94%A8%E5%B7%A5%E5%85%B7)
        - [Gdbserver](#gdbserver)
        - [Strace](#strace)
    - [有源码so调试](#%E6%9C%89%E6%BA%90%E7%A0%81so%E8%B0%83%E8%AF%95)
        - [Ndk-gdb](#ndk-gdb)
        - [Gdb-Gdbserver](#gdb-gdbserver)
    - [无源码调试So](#%E6%97%A0%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95so)
        - [使用Arm版Gdb在移动端直接调试](#%E4%BD%BF%E7%94%A8arm%E7%89%88gdb%E5%9C%A8%E7%A7%BB%E5%8A%A8%E7%AB%AF%E7%9B%B4%E6%8E%A5%E8%B0%83%E8%AF%95)
        - [IDA调试](#ida%E8%B0%83%E8%AF%95)
        - [Gdb-Gdbserver](#gdb-gdbserver)
        - [Gikdbg](#gikdbg)
            - [调试so](#%E8%B0%83%E8%AF%95so)
    - [调试Android上Linux程序](#%E8%B0%83%E8%AF%95android%E4%B8%8Alinux%E7%A8%8B%E5%BA%8F)
        - [技巧：如何在so入口下断?](#%E6%8A%80%E5%B7%A7%EF%BC%9A%E5%A6%82%E4%BD%95%E5%9C%A8so%E5%85%A5%E5%8F%A3%E4%B8%8B%E6%96%AD)
- [Java层/Linux层联合调试](#java%E5%B1%82linux%E5%B1%82%E8%81%94%E5%90%88%E8%B0%83%E8%AF%95)
    - [有源码联合调试](#%E6%9C%89%E6%BA%90%E7%A0%81%E8%81%94%E5%90%88%E8%B0%83%E8%AF%95)
    - [无源码联合调试](#%E6%97%A0%E6%BA%90%E7%A0%81%E8%81%94%E5%90%88%E8%B0%83%E8%AF%95)
        - [操作步骤](#%E6%93%8D%E4%BD%9C%E6%AD%A5%E9%AA%A4)
        - [最简单的gdb中，断在加载so时刻的方法](#%E6%9C%80%E7%AE%80%E5%8D%95%E7%9A%84gdb%E4%B8%AD%EF%BC%8C%E6%96%AD%E5%9C%A8%E5%8A%A0%E8%BD%BDso%E6%97%B6%E5%88%BB%E7%9A%84%E6%96%B9%E6%B3%95)
    - [Android linux内核层调试](#android-linux%E5%86%85%E6%A0%B8%E5%B1%82%E8%B0%83%E8%AF%95)
- [使用Hook](#%E4%BD%BF%E7%94%A8hook)
    - [常用Hook/Inject工具简介](#%E5%B8%B8%E7%94%A8hookinject%E5%B7%A5%E5%85%B7%E7%AE%80%E4%BB%8B)
- [实例 360手机卫士卸载后弹窗分析过程](#%E5%AE%9E%E4%BE%8B-360%E6%89%8B%E6%9C%BA%E5%8D%AB%E5%A3%AB%E5%8D%B8%E8%BD%BD%E5%90%8E%E5%BC%B9%E7%AA%97%E5%88%86%E6%9E%90%E8%BF%87%E7%A8%8B)
    - [现象](#%E7%8E%B0%E8%B1%A1)
    - [文件注入](#%E6%96%87%E4%BB%B6%E6%B3%A8%E5%85%A5)
        - [分析日志](#%E5%88%86%E6%9E%90%E6%97%A5%E5%BF%97)
        - [定位关键代码](#%E5%AE%9A%E4%BD%8D%E5%85%B3%E9%94%AE%E4%BB%A3%E7%A0%81)
        - [结论](#%E7%BB%93%E8%AE%BA)
- [GDB调试](#gdb%E8%B0%83%E8%AF%95)
    - [反汇编一段地址](#%E5%8F%8D%E6%B1%87%E7%BC%96%E4%B8%80%E6%AE%B5%E5%9C%B0%E5%9D%80)
    - [表达式计算](#%E8%A1%A8%E8%BE%BE%E5%BC%8F%E8%AE%A1%E7%AE%97)
    - [查看寄存器](#%E6%9F%A5%E7%9C%8B%E5%AF%84%E5%AD%98%E5%99%A8)
    - [查看栈参数](#%E6%9F%A5%E7%9C%8B%E6%A0%88%E5%8F%82%E6%95%B0)
    - [查看局部变量](#%E6%9F%A5%E7%9C%8B%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F)
    - [查看内存](#%E6%9F%A5%E7%9C%8B%E5%86%85%E5%AD%98)
    - [修改内存](#%E4%BF%AE%E6%94%B9%E5%86%85%E5%AD%98)
    - [断点](#%E6%96%AD%E7%82%B9)
        - [一次断点](#%E4%B8%80%E6%AC%A1%E6%96%AD%E7%82%B9)
        - [条件/线程断点](#%E6%9D%A1%E4%BB%B6%E7%BA%BF%E7%A8%8B%E6%96%AD%E7%82%B9)
        - [观察断点](#%E8%A7%82%E5%AF%9F%E6%96%AD%E7%82%B9)
        - [范围断点](#%E8%8C%83%E5%9B%B4%E6%96%AD%E7%82%B9)
        - [硬件断点](#%E7%A1%AC%E4%BB%B6%E6%96%AD%E7%82%B9)
        - [拦截当前函数退出](#%E6%8B%A6%E6%88%AA%E5%BD%93%E5%89%8D%E5%87%BD%E6%95%B0%E9%80%80%E5%87%BA)
        - [捕获断点](#%E6%8D%95%E8%8E%B7%E6%96%AD%E7%82%B9)
        - [跟踪断点](#%E8%B7%9F%E8%B8%AA%E6%96%AD%E7%82%B9)
    - [查看调用栈](#%E6%9F%A5%E7%9C%8B%E8%B0%83%E7%94%A8%E6%A0%88)
    - [流程控制](#%E6%B5%81%E7%A8%8B%E6%8E%A7%E5%88%B6)
        - [反向调试](#%E5%8F%8D%E5%90%91%E8%B0%83%E8%AF%95)
    - [显示当前加载模块](#%E6%98%BE%E7%A4%BA%E5%BD%93%E5%89%8D%E5%8A%A0%E8%BD%BD%E6%A8%A1%E5%9D%97)
    - [强制加载模块](#%E5%BC%BA%E5%88%B6%E5%8A%A0%E8%BD%BD%E6%A8%A1%E5%9D%97)
    - [替换当前调试模块](#%E6%9B%BF%E6%8D%A2%E5%BD%93%E5%89%8D%E8%B0%83%E8%AF%95%E6%A8%A1%E5%9D%97)
    - [进程转储](#%E8%BF%9B%E7%A8%8B%E8%BD%AC%E5%82%A8)
    - [进程空间](#%E8%BF%9B%E7%A8%8B%E7%A9%BA%E9%97%B4)
    - [由地址获对应的符号](#%E7%94%B1%E5%9C%B0%E5%9D%80%E8%8E%B7%E5%AF%B9%E5%BA%94%E7%9A%84%E7%AC%A6%E5%8F%B7)
    - [查找符号](#%E6%9F%A5%E6%89%BE%E7%AC%A6%E5%8F%B7)
    - [执行外部命令](#%E6%89%A7%E8%A1%8C%E5%A4%96%E9%83%A8%E5%91%BD%E4%BB%A4)
    - [显示线程](#%E6%98%BE%E7%A4%BA%E7%BA%BF%E7%A8%8B)
    - [打印c++对象虚表](#%E6%89%93%E5%8D%B0c%E5%AF%B9%E8%B1%A1%E8%99%9A%E8%A1%A8)

<!-- /TOC -->


# 概述

## 分析步骤

### 通用逆向分析步骤

* 1.了解该模块正向编程相关方法 
* 2.使用apktool解密apk，得到资源、jni模块等文件
* 3.从apk提取出dex文件，使用dex2jar转换成jar文件，再用java逆向工具得到java源码 dex->jar->java
* 4.根据特征(字符串、常量、包名类名方法名、manifest文件、布局文件等方式)或调试手段定位到关键代码
* 5.分析变量含义类型、函数逻辑、模块流程 
* 6.对变量、函数、类进行标注、恢复成高级语言	->c

&emsp;&emsp;Android程序的特点相比在于使用混淆方式打包，将包名、类名、函数名改成不易看懂的字母，从而使生成的apk小很多(android studio提供了release编译方式，使用proguard混淆)，因此反编译apk最多的工作在于重构这些名称，这一点和pc上一致，对于android native程序(jni)则和pc上基本一致，不同之处在于常见的是arm汇编。

### 安卓上APK调试步骤：
* 1.Apk(debuggable)或系统(ro.debuggable=1)设置为可调试
* 2.在虚拟机中启动服务端(adbd/android_server)
* 3.在主机端连接客户端调试器(IDA/jdb/adt)，设置断点

### 安卓上linux程序调试步骤：
* 1.在虚拟机中启动服务端(gdb_server/linux_server)
* 2.在主机端连接客户端调试器(IDA/gdb_for_windows)，设置断点

&emsp;&emsp;对于apk的反编译，由于资源和xml都进行了编码，因此反编译时必然要解析相应的resource.arsc/AndroidManifest.xml等文件，对于做过保护处理的apk通常会在这里做手脚干扰Apktool、dex2jar等反编译工具因此很有必要掌握编译、调试这些工具源码的方法(见“如何编译、调试apktool和dex2jar”)

## 分析工具

* 集成IDE：APK改之理、JD-GUI、JEB(1.4破解 2.0)、jadx
* 解压(apk, jar)：WinRar 
* 解析资源：apktool 
* 反编译引擎(jar, class)：dex2jar工具集、jd-core(JD-GUI,JD-Eclipse反编译核心)、fernflower(Android Studio反编 、procyon 
* 回编译：aapt、dex2jar工具集
* 调试器：IDA、jdb、adt等
* 辅助工具：DDMS  如果是虚拟机可以看到所有进程

### APK改之理
* 整合&提供了全套解压、反编译代码和资源、回编译、签名功能，强大的正则搜索，修改smali字节码等功能
* 集成ApkTool、Dex2jar、JD-GUI工具
* 可视化操作，全自动的反编译、回编译、签名Apk 
* 正则表达式搜索资源及源码 

### JD-GUI
轻量级反编译，反编译jar/class等java字节码文件(能力一般)，提供简单的搜索能力

### JEB
* 反编译apk/jar工具(能力较强)
* 强大的正向、反向索引，一定程度重命名能力，一定搜索能力
* 支持注释、插件 
* 交互式可视化操作，全自动的反编译 
* 支持重命名

### Dex2jar工具集
&emsp;&emsp;dex2jar是一个工具包，反编译dex和jar，还提供了一些其它的功能，每个功能使用一个bat批处理或 sh 脚本来包装，只需在Windows 系统中调用 bat文件、在Linux 系统中调用 sh 脚本即可。在bat中调用相应的jar主类完成特定功能，例如d2j-dex2jar.bat中的内容是：`@"%~dp0d2j_invoke.bat" com.googlecode.dex2jar.tools.Dex2jarCmd %*`。常用的有`dex2jar jar2dex dex2smali smali2dex`  
* d2j-apk-sign用来为apk 文件签名。命令格式：d2j-apk-sign xxx.apk 。  
* d2j-asm-verify 用来验证jar 文件。命令格式：d2j-asm-verify -d xxx.jar。  
* d2j-dex2jar 用来将dex 文件转换成jar 文件。命令格式：d2j-dex2jar xxx.apk  
* d2j-dex-asmifier 用来验证dex 文件。命令格式：d2j-dex-asmifier xxx.dex。  
* d2j-dex-dump 用来转存dex 文件的信息。命令格式：d2j-dex-dump xxx.apk out.jar 。  
* d2j-init-deobf 用来生成反混淆jar 文件时的初始化配置文件。  
* d2j-jar2dex 用来将jar 文件转换成 dex 文件。命令格式：d2j-jar2dex xxx.apk。  
* d2j-jar2jasmin 用来将jar 文件转换成jasmin 格式的文件。命令格式：d2j-jar2jasmin xxx.jar  
* d2j-jar-access 用来修改jar 文件中的类、方法以及字段的访问权限。  
* d2j-jar-remap 用来重命名jar 文件中的包、类、方法以及字段的名称。  
* d2j-jasmin2jar 用来将jasmin 格式的文件转换成 jar 文件。命令格式：d2j-jasmin2jar dir dex2jar为d2j-dex2jar 的副本。  
* dex-dump为d2j-dex-dump 的副本  

### Apktool反编译&打包工具
* 反编译apk：apktool d file.apk –o path  
* 回编译apk：apktool b path –o file.apk  

## 常见文件格式

### Apk
&emsp;&emsp;Android package，android安装程序文件，本质上是压缩包，解压得到classes.dex、resources.arsc、AndroidManifest.xml、so文件以及资源文件
* Resources.arsc资源描述文件
* Classes.dex所有代码编译过得darvik字节码文件，可能会有多个
* AndroidManifest.xml 编译过的AndroidManifest.xml文件

#### 使用aapt解析xml
```
aapt d xmltree 1.apk AndroidManifest.xml
N: android=http://schemas.android.com/apk/res/android
  E: manifest (line=2)
    A: android:versionCode(0x0101021b)=(type 0x10)0x1
    A: android:versionName(0x0101021c)="1.0" (Raw: "1.0")
    A: package="com.ibotpeaches.issue767" (Raw: "com.ibotpeaches.issue767")
    A: platformBuildVersionCode=(type 0x10)0x17 (Raw: "23")
    A: platformBuildVersionName="6.0-2438415" (Raw: "6.0-2438415")
    E: uses-sdk (line=0)
      A: android:minSdkVersion(0x0101020c)=(type 0x10)0x16
      A: android:targetSdkVersion(0x01010270)=(type 0x10)0x17
    E: application (line=3)
      A: android:theme(0x01010000)=@0x7f090083
      A: android:label(0x01010001)=@0x7f060015
      A: android:icon(0x01010002)=@0x7f030000
      A: android:debuggable(0x0101000f)=(type 0x12)0xffffffff
      A: android:allowBackup(0x01010280)=(type 0x12)0xffffffff
      A: android:supportsRtl(0x010103af)=(type 0x12)0xffffffff
      E: activity (line=4)
        A: android:theme(0x01010000)=@0x7f090030
        A: android:label(0x01010001)=@0x7f060015
        A: android:name(0x01010003)="com.ibotpeaches.issue767.MainActivity" (Raw
: "com.ibotpeaches.issue767.MainActivity")
        E: intent-filter (line=5)
          E: action (line=6)
            A: android:name(0x01010003)="android.intent.action.MAIN" (Raw: "andr
oid.intent.action.MAIN")
          E: category (line=7)
            A: android:name(0x01010003)="android.intent.category.LAUNCHER" (Raw:
 "android.intent.category.LAUNCHER")
      E: meta-data (line=10)
        A: android:name(0x01010003)="large.int.value" (Raw: "large.int.value")
        A: android:value(0x01010024)="9999999999999999999999" (Raw: "99999999999
99999999999")
```

查看xml  =>  aapt d xmltree 1.apk AndroidManifest.xml  
查看resource  =>  aapt d resources 1.apk     (resource.arsc)

### Dex
&emsp;&emsp;Dalvik Executable，Dalvik可执行文件，从java class文件转换而来的字节码，Classes.Dex通过dex2jar转换成java字节码(有损)，或者dex2smali转换成darvik汇编(无损)——smali字节码，其形式如下
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_18.png)

### Jar
&emsp;&emsp;Java Archive，java归档文件，可以直接解压得到class文件

### Odex
dex转odex：/system/bin/dexopt  
dexopt-wrapper 1.apk 1.odex  

### Aar
&emsp;&emsp;Android归档文件，压缩包格式，包含  
* /AndroidManifest.xml (强制)		未编译的
* /classes.jar (强制)
* /res/ (强制)
* /R.txt (强制)
* /assets/ (可选)
* /libs/*.jar (可选)
* /jni/<abi>/*.so (可选)
* /proguard.txt (可选)
* /lint.jar (可选)

### So
&emsp;&emsp;Linux动态链接库文件，包含arm64 arm mips mips64 x86 x86-64几个平台

### 工具转换图
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_19.png)

### Android设备上重要目录
* /system/app/1.apk  系统应用
* /data/app/1.apk    用户应用
* /data/data/[pkgname]  应用数据(so,database,…)
* /data/dalvik-cache 存放dex

# Java层

## 常用工具

### adb
&emsp;&emsp;设备通信、调试工具，常用法：
```
adb devices 列出当前设备
adb –s d24eb3ab [命令]      指定设备执行命令
adb push 源 目标			 非root机器可以设置路径为/data/local/tmp
adb pull 源 目标
adb shell 				    执行终端
adb logcat 				    查看日志(/system/logcat为服务器)
adb jdwp 				    查看远程jdwp进程
adb forward tcp:主机端口     tcp:远程端口 		把主机端口消息转发手机端口(端口对应进程)	用于ida调试
adb forward tcp:主机端口     jdwp:远程进程ID 	把主机端口消息转发手机jdwp进程	用于jdb调试 
adb install [apkpath]		安装apk
adb uninstall [packagename]	卸载apk 注意会彻底清理，删除/data/app下的备份apk
adb remount 				将/system重新映射为读写，以便进行系统区文件操作
adb root                    使adb以root方式启动，便于push/pull/remount
```

### aapt
&emsp;&emsp;APK资源管理工具，用于增删查改APK中的文件、资源等，对于分析编译后的Resource.arsc, AndroidManifest.xml格式较有价值，通常也可以用winrar对apk/jar进行解压
```
打印xml树 aapt d xmltree 1.apk AndroidManifest.xml
打印资源	aapt d resources 1.apk
添加文件	aapt a 1.apk AndroidManifest.xml
删除文件	aapt r 1.apk AndroidManifest.xml
```

### am & pm
&emsp;&emsp;Android远程命令，am执行调试、运行功能，pm执行安装、卸载功能

* 启动应用：`am start -D -n "b.myapp/b.myapp.MainActivity" -a android.intent.action.MAIN -c  android.intent.category.LAUNCHER`
* 启动服务：`am startservice -n com.android.music/com.android.music.MediaPlaybackService`
* 强制停止包：`am force-stop com.example.administrator.myapplication`
* 强制结束包进程：`am kill com.example.administrator.myapplication`      `am kill all`
* 发送广播：`adb shell am broadcast -a com.android.test`
* 安装应用：`pm install –r 1.apk`
* 卸载应用：`pm uninstall packagename`
* 列出所有安装包：`pm list package`
* 查看是否以指定名为前缀的包存在：`pm list package com.qihoo`
* 禁用应用：`pm disable packagename`  (禁用后，图标消失，对该应用的操作都无效)

## 有源码调试APK

### Android studio
&emsp;&emsp;在android studio中可以采用运行调试或进程附加方式调试，支持条件断点、一次断点、对单线程下断，有6种断点：

|TypeCh               |TypEn                    |Description                                |
|---------------------|-------------------------|-------------------------------------------|
|行断点               |Java Line Breakpoints    |在(java/c)源码某行下断                       |
|Java类成员变量访问断点|Java Field Watchpoints   |类似于内存访问断点，在读和写java类成员变量时断下|
|Java类方法断点       |Java Method Breakpoints   |在进入java层函数或退出函数时断下              |
|Java异常断点         |Java Exception Breakpoints| 发生java层捕获或未捕获异常时断下             |
|异常断点             |Exception Breakpoints     |抛异常或捕获异常时断下                       |
|符号断点             |Symbolic Breakpoints      |(c/java)符号断点                            |


### Adb wifi 
&emsp;&emsp;应用市场有很多这种软件，需要Root权限。解决没有USB数据线的情况下的调试
```
C:\Users\Administrator>adb connect 192.168.0.103:5555
connected to 192.168.0.103:5555
此时可以用adt调试
```

## 无源码调试apk
&emsp;&emsp;不需要调试的一般过程 ：使用反编译工具得到源代码，修改调试标识，修改机器码，最后回编译签名：
```
反编译apk：apktool d file.apk –o path  
回编译apk：apktool b path –o file.apk  
```

### 使用AndroidStudio和Apktool工具调试
* 第一步，反编译得到(占行)伪源码：`java -jar apktool.jar d -d input.apk -o out`，加上-d选项之后反编译出的文件后缀为.java，而不是.smali，每个.java文件立马都伪造成了一个类，语句全都是“a=0;”这一句，smali语句成为注释，做这些都是为了后面欺骗idea、eclipse、android studio这些ide的
* 第二步，修改资源或者源码(smali)，修改AndroidManifest.xml调试标识，反编译以后可以在dex中插入waitfordebugger或者Log.i的smali代码来进行相应的控制
* 第三步，回编译(-d选项)+签名 
* * 回编译：`apktool b –d path –o input.apk`
* * 签名：	`java –jar signapk.jar testkey.x509.pem testkey.pk8  input.apk output.apk`
* 第四步，新建android studio工程 ，将反编译得到的smali文件夹中的源文件拷贝到源码目录（欺骗），回编译的apk覆盖目标apk位置 ，删除Edit configuration的Before launch，下断点调试 

***点评：这种方式只可以用来分析加密很弱的App，前提是apktool可以成功反编译***  
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_20.png)

### 使用jdb调试
&emsp;&emsp;jdb是一个支持java代码级调试的工具，它是由java jdk提供的，可以设置断点、查看堆栈、计算表达式、动态修改类字节码、调试&跟踪、修改变量值、线程操作，断点包括：(源码)行断点、符号断点、成员变量访问断点。每个java程序(windows/ios/android)都可以用jdwp协议进行调试，Android Studio/Eclipse的调试也是建立在该协议基础之上，下面以实例说明：

* 第一步，开发demo
```Java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.ok).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse("http://www.b.com"));
                intent.setClassName("com.android.browser","com.android.browser.BrowserActivity");
                startActivity(intent);
            }
        });
    }
}
```

* 第二步，启动jdb调试
```
adb shell am start -D -n "b.myapp/b.myapp.MainActivity" -a android.intent.action.MAIN -c android.intent.category.LAUNCHER
```
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_21.png)

* 第三步，开始调试
* * 查看ddms中该进程端口号 8600  
* * 使用jdb调试：jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8600
* * 下断点：函数断点stop in android.app.Activity.startActivity(android.content.Intent)  
行断点 stop at android.app.Activity:123，触发断点后显示堆栈：
```
<1> main[1] where
  [1] android.app.Activity.startActivity (Activity.java:3,490)
  [2] b.myapp.MainActivity$1.onClick (MainActivity.java:21)
  [3] android.view.View.performClick (View.java:4,084)
  [4] android.view.View$PerformClick.run (View.java:16,966)
  [5] android.os.Handler.handleCallback (Handler.java:615)
  [6] android.os.Handler.dispatchMessage (Handler.java:92)
  [7] android.os.Looper.loop (Looper.java:137)
  [8] android.app.ActivityThread.main (ActivityThread.java:4,745)
  [9] java.lang.reflect.Method.invokeNative (本机方法)
```

#### 查看参数
```
<1> main[1] print intent
 intent = "Intent { act=android.intent.action.VIEW dat=http://www.b.com cmp=
com.android.browser/.BrowserActivity }"
```

#### 设置源码从而进行逐行调试
```
<1> main[1] use D:\Android\sdk\sources\android-18		//参考设备android版本
<1> main[1] use D:\test\MyApplication\app\src\main\java
<1> main[1] list
3,421         * @hide Implement to provide correct calling token.
3,422         */
3,423        public void startActivityAsUser(Intent intent, UserHandle user) {
3,424            startActivityAsUser(intent, null, user);
3,425 =>     }
3,426
3,427        /**
3,428         * @hide Implement to provide correct calling token.
3,429         */
3,430        public void startActivityAsUser(Intent intent, Bundle options, User
Handle user) {
```

#### 行断点：
```
> use D:\test\MyApplication\app\src\main\java
stop at b.myapp.MainActivity:18
正在延迟断点b.myapp.MainActivity:18。
将在加载类后设置。
> resume
已恢复所有线程。
> 设置延迟的断点b.myapp.MainActivity:18
断点命中: "线程=<1> main", b.myapp.MainActivity.onCreate(), 行=18 bci=12
18            int j = 0;
```

#### 初始断点
&emsp;&emsp;只要连接到jdb就会导致app运行起来，此时如果想断在初始化这部分就没有办法了，不过jdb提供初始命令脚本
* 暂停所有线程： `echo suspend > jdb.ini`
* 执行调试：`jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8601`  

&emsp;&emsp;此时，app仍然处于等调试器状态，而虫子已经变绿，此时可以下断点，然后resume恢复所有线程
附加后会变绿色虫子
```
> > stop in b.myapp.MainActivity.onCreate(android.os.Bundle)
正在延迟断点b.myapp.MainActivity.onCreate(android.os.Bundle)。
将在加载类后设置。
>resume
已恢复所有线程。
断点命中: "线程=<1> main", b.myapp.MainActivity.onCreate(), 行=13 bci=0

<1> main[1] where
  [1] b.myapp.MainActivity.onCreate (MainActivity.java:13)
  [2] android.app.Activity.performCreate (Activity.java:5,372)
  [3] android.app.Instrumentation.callActivityOnCreate (Instrumentation.java:1,1
04)
  [4] android.app.ActivityThread.performLaunchActivity (ActivityThread.java:2,25
8)
  [5] android.app.ActivityThread.handleLaunchActivity (ActivityThread.java:2,350
)
  [6] android.app.ActivityThread.access$700 (ActivityThread.java:160)
  [7] android.app.ActivityThread$H.handleMessage (ActivityThread.java:1,317)
  [8] android.os.Handler.dispatchMessage (Handler.java:99)
  [9] android.os.Looper.loop (Looper.java:137)
  [10] android.app.ActivityThread.main (ActivityThread.java:5,454)
```

#### 调试命令
```
	stop in:断点
	step:步入(源码行)
	stepi:单入(指令)
	step up:执行到返回
	cont:恢复运行
	next:步过
	输出表达式:print/eval
```

***jdb最大缺点在于难用，所以有人用python封装了一次，工具名AndBug***

## 无源码调试dex
* 1.	使用ida分析apk或者从apk中提取出的dex 
* 2.	设置调试选项，包括包名和主类名，参考反编译的AndroidManifest
* 3.	启动调试即可  
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_22.png)  
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_23.png)  

# Linux层

## 常用工具

### Gdbserver
```
Usage:  gdbserver [OPTIONS] COMM PROG [ARGS ...]
        gdbserver [OPTIONS] --attach COMM PID
        gdbserver [OPTIONS] --multi COMM
		隐藏用法：gdbserver [OPTIONS] +SOCKETFILE --attach PID			会在本地建立socket文件通信
Options:
  --debug               Enable general debugging output.
  --remote-debug        Enable remote protocol debugging output.
  --version             Display version information and exit.
  --wrapper WRAPPER --  Run WRAPPER to start new programs.
  --once                Exit after the first connection has closed.
使用方式：
启动模式远程调试：gdbserver --debug --remote-debug  :23946 /system/test.out [参数]		
附加模式远程：gdbserver –debug –remote-debug –attach  :23946 1234
	Adb forward tcp:23946 tcp:23946 转发端口
IDA中选择Remote GDB Debugger附加即可
```

### Strace
```
usage: strace [-CdDffhiqrtttTvVxx] [-a column] [-e expr] ... [-o file]
              [-p pid] ... [-s strsize] [-u username] [-E var=val] ...
              [command [arg ...]]
or: strace -c [-D] [-e expr] ... [-O overhead] [-S sortby] [-E var=val] ...
              [command [arg ...]]
-c --统计每一系统调用的所执行的时间,次数和出错的次数等.
-C -- like -c but also print regular output while processes are running
-f --跟踪由fork调用所产生的子进程.
-F --尝试跟踪vfork调用.在-f时,vfork不被跟踪.
-i --输出系统调用的入口指针
-q --禁止输出关于脱离的消息
-r --打印出相对时间关于,,每一个系统调用
-T --显示每一调用所耗的时间
-v --输出所有的系统调用.一些调用关于环境变量,状态,输入输出等调用由于使用频繁,默认不输出
-x --以十六进制形式输出非标准字符串
-a设置返回值的输出位置.默认 为40.
-e expr -指定一个表达式,用来控制如何跟踪.: option=[!]all or option=[!]val1[,val2]...
   options: trace, abbrev, verbose, raw, signal, read, or write
-o file --将strace的输出写入文件filename
-O overhead -- set overhead for tracing syscalls to OVERHEAD usecs
-p pid --跟踪指定的进程pid.
-D -- run tracer process as a detached grandchild, not as parent
-s strsize --指定输出的字符串的最大长度.默认为32.文件名一直全部输出
-S sortby -- sort syscall counts by: time, calls, name, nothing (default time)
-u username --以username 的UID和GID执行被跟踪的命令
-E var=val -- put var=val in the environment for command
-E var -- remove var from the environment for command

使用方式：
Strace –f ProcessA		启动跟踪
Strace –f –p 234		附加跟踪
	-e trace=file       -e trace=process    -e trace=network
```

## 有源码so调试

### Ndk-gdb
&emsp;&emsp;该程序是一个shell脚本，执行过程如下：
```
adb shell am start -D -n com.example.hellojni/.HelloJni		启动app并等待调试器
	ps | grep hellojni									得到PID 3569
adb shell run-as com.example.hellojni /data/data/com.example.hellojni/lib/gdbserver +debug-socket --attach (3569)PID
	将PID与文件映射建立调试链接(c层)
adb forward tcp:5039 localfilesystem:/data/data/com.example.hellojni/debug-socket将调试链接和本地端口建立链接(c层)
adb forward tcp:65534 jdwp:(3569)PID										 将本地端口和进程建立连接(java层)
jdb -connect com.sun.jdi.SocketAttach:hostname=localhost,port=65534			 使用jdb调试java层
arm-linux-androideabi-gdb.exe											
	target remote :5039												 使用gdb调试c层
	set breakpoint pending on
```

&emsp;&emsp;(使用前关掉腾讯的AndroidServer.exe，否则连不上!!!)，在工程目录下(有AndroidManifest.xml)，命令行运行`%NDK_ROOT%\ndk-gdb-py.cmd --start --verbose`，输出下面字符即为成功：
```
Android NDK installation path: D:/Android/AndroidNDK/android-ndk-r10e
ADB version found: Android Debug Bridge version 1.0.32
Using ADB flags:
Using auto-detected project path: .
Found package name: com.example.hellojni
ABIs targetted by application: arm64-v8a armeabi armeabi-v7a armeabi-v7a-hard mips mips64 x86 x86_64
Device API Level: 19
Device CPU ABIs: armeabi-v7a armeabi
Compatible device ABI: armeabi-v7a
Using gdb setup init: ./libs/armeabi-v7a/gdb.setup
Using toolchain prefix: D:/Android/AndroidNDK/android-ndk-r10e/toolchains/arm-linux-androideabi-4.8/prebuilt/windows/bin/arm-linux-androideabi
Using app out directory: ./obj/local/armeabi-v7a
Found debuggable flag: true
Found device gdbserver: /data/data/com.example.hellojni/lib/gdbserver
Found data directory: '/data/data/com.example.hellojni'
Found first launchable activity: .HelloJni
Launching activity: com.example.hellojni/.HelloJni
## COMMAND: adb_cmd shell am start -D -n com.example.hellojni/.HelloJni
## COMMAND: adb_cmd shell sleep 2.000000
Found running PID: 9139
## COMMAND: adb_cmd shell run-as com.example.hellojni /data/data/com.example.hellojni/lib/gdbserver --attach +debug-socket 9139 [BACKGROUND]
Launched gdbserver succesfully.
Setup network redirection
## COMMAND: adb_cmd forward tcp:5039 localfilesystem:/data/data/com.example.hellojni/debug-socket
Attached; pid = 9139
Listening on Unix socket debug-socket
## COMMAND: adb_cmd pull /system/bin/app_process ./obj/local/armeabi-v7a/app_process
79 KB/s (9488 bytes in 0.117s)
Pulled app_process from device/emulator.
## COMMAND: adb_cmd pull /system/bin/linker ./obj/local/armeabi-v7a/linker
585 KB/s (63596 bytes in 0.106s)
Pulled linker from device/emulator.
## COMMAND: adb_cmd pull /system/lib/libc.so ./obj/local/armeabi-v7a/libc.so
1184 KB/s (310584 bytes in 0.256s)
Pulled /system/lib/libc.so from device/emulator.
Set up JDB connection, using jdb command: C:\Program Files\Java\jdk1.8.0_66\bin\jdb.exe
## COMMAND: adb_cmd forward tcp:65534 jdwp:9139
--------------------./obj/local/armeabi-v7a/gdb.setup---------------
GNU gdb (GDB) 7.7
Copyright (C) 2014 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=i586-pc-mingw32msvc --target=arm-linux-android".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://source.android.com/source/report-bugs.html>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word".
Remote debugging from host 9.11.5.0
warning: Could not load shared library symbols for 112 libraries, e.g. libstdc++.so.
Use the "info sharedlibrary" command to see the complete listing.
Do you need "set solib-search-path" or "set sysroot"?
0x400daa80 in __futex_syscall3 () from D:\Android\AndroidNDK\android-ndk-r10e\samples\hello-jni\obj\local\armeabi-v7a\libc.so
(gdb)
```

***点评：该工具要求环境极为苛刻且不稳定，不建议使用***

### Gdb-Gdbserver
&emsp;&emsp;操作步骤：
* Android studio导入jni工程,
* 拷贝.so到搜索路径，pull /system/lib到搜索路径，pull /system/linker到搜索路径
* 启动gdbserver (具体命令根据版本不同而变)  
`gdbserver --attach *:111 1234`
* 转发端口  
`adb forward tcp:111 tcp:111`
* 连接本地调试器  
`target remote 127.0.0.1:111`

```
(gdb) set solib-search-path C:/Users/lichao/2/
Reading symbols from C:\Users\lichao\2\linker...(no debugging symbols found)...done.
Loaded symbols for C:\Users\lichao\2\linker
Reading symbols from C:\Users\lichao\2\libc.so...(no debugging symbols found)...done.
Loaded symbols for C:\Users\lichao\2\libc.so
Reading symbols from C:\Users\lichao\2\libstdc++.so...(no debugging symbols found)...done.
Loaded symbols for C:\Users\lichao\2\libstdc++.so
Reading symbols from C:\Users\lichao\2\libm.so...(no debugging symbols found)...done.
Loaded symbols for C:\Users\lichao\2\libm.so
Reading symbols from C:\Users\lichao\2\liblog.so...(no debugging symbols found)...done.
Loaded symbols for C:\Users\lichao\2\liblog.so
Reading symbols from C:\Users\lichao\2\libcutils.so...(no debugging symbols found)...done.
Loaded symbols for C:\Users\lichao\2\libcutils.so
Reading symbols from C:\Users\lichao\2\libgccdemangle.so...(no debugging symbols found)...done.
Loaded symbols for C:\Users\lichao\2\libgccdemangle.so
Reading symbols from C:\Users\lichao\2\libcorkscrew.so...(no debugging symbols found)...done.

(gdb) bt
#0  0x400e50e0 in fork () from C:\Users\lichao\2\libc.so
#1  0x76886ca0 in Java_com_example_hellojni_HelloJni_stringFromJNI () from C:\Users\lichao\2\libhello-jni.so
#2  0x416b8350 in dvmPlatformInvoke () from C:\Users\lichao\2\libdvm.so
#3  0x416e8fd2 in dvmCallJNIMethod(unsigned int const*, JValue*, Method const*, Thread*) () from C:\Users\lichao\2\libdvm.so
#4  0x416ea9ba in dvmResolveNativeMethod(unsigned int const*, JValue*, Method const*, Thread*) () from C:\Users\lichao\2\libdvm.so
#5  0x416c1828 in dvmJitToInterpNoChain () from C:\Users\lichao\2\libdvm.so
Backtrace stopped: previous frame identical to this frame (corrupt stack?)
(gdb) list
```

## 无源码调试So

### 使用Arm版Gdb在移动端直接调试
* 获取arm版gdb
* 把gdb下载到移动端  
`adb push gdb /data/bin`
* 执行gdb  
`adb shell`  
`./data/bin/gdb`  

```
GNU gdb 6.7
Copyright (C) 2007 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=arm-none-linux-gnueabi --target=".
(gdb) 
```
***点评：该方法速度快，但不好查看符号***

### IDA调试
* 将android_server拷入/data/local/tmp/      
`adb push  android_server /data/local/tmp/`
* 修改可执行权限，运行   
`cd /data/local/tmp/`  
`chmod 755 android_server`  
`./android_server`  
* 将模拟器端口转发至pc端口 (另开启命令行)  
`adb forward tcp:23946 tcp:23946`
* IDA中选择`Remote ARMLinux/Android debugger`，端口23946，调试即可，成功以后显示  
`Accepting connection from 127.0.0.1...`

### Gdb-Gdbserver
* 启动server  
`./gdbserver –attach :1234 [pid]`
* 转发端口  
`adb forward tcp:1234 tcp:1234`
* 启动client  
`arm-linux-androideabi-gdb.exe`
* 连接server  
`target remote :1234`
* 设置单步调试  
`set step-mode on`			
* 设置反汇编模式  
`set disassemble-next on`
* 设置加载so断点  
`catch load 1.so`

```
0xb6cdf480 in __epoll_pwait () from E:\aaa\libc.so
=> 0xb6cdf480 <__epoll_pwait+28>:       1e ff 2f 91     bxls    lr
(gdb) bt
#0  0xb6cdf480 in __epoll_pwait () from E:\aaa\libc.so
#1  0xb6cb70ca in epoll_pwait () from E:\aaa\libc.so
#2  0xb6cb70d8 in epoll_wait () from E:\aaa\libc.so
#3  0xb6f06bd6 in android::Looper::pollInner(int) () from E:\aaa\libutils.so
#4  0xb6f06e52 in android::Looper::pollOnce(int, int*, int*, void**) () from E:\aaa\libutils.so
#5  0xb6e4d41c in android::NativeMessageQueue::pollOnce(_JNIEnv*, _jobject*, int) () from E:\aaa\libandroid_runtime.so
#6  0x732e056e in ?? ()
```

### Gikdbg
&emsp;&emsp;GikDbg 是一款移动平台的汇编级调试器，它基于 OllyDbg ，GDB 以及 LLVM 实现而来。OllyDbg 现已广泛用于 PC 平台软件安全领域，GikDbg 是 OllyDbg 向移动平台转移的产物，它可以协助您完成诸如应用调试分析，应用安全评估，应用漏洞挖掘等移动安全领域。What features can GikDbg support?  <http://gikir.com/product.php>
* ELF / Mach-O executable file static analysis;
* Android / iOS App dynamic debugging;
* Android / iOS remote console;
* ARM assembler；
* ARM disassembler；
* Device file uploading and downloading；
* Built-in GDB and LLDB；
* Support for memory breakpoint, software breakpoint, conditional breakpoint;
* Support for multi-threaded debugging;
* Support for assembly code level file patching.

GikDbg for IOS  
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_25.png)

GikDbg for Android  
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_26.png)

&emsp;&emsp;gikdbg.art-Gikir Debugger for Android RunTime, 是Android平台的32位汇编级调试器。此处的Android RunTime既指DVM RunTime又指ART RunTime，因此不管是运行dalvik虚拟机还是运行本地代码的art均可以使用gikdbg.art进行程序的二进制调试分析。不同之处在于dalvik虚拟机的运行时只能调试so动态库，而art运行时不仅能调试so动态库，还能调试系统镜像oat，可执行程序dex这样的文件。另外，gikdbg-Gikir Debugger for iPhone OS，是调试越狱苹果设备的32位汇编级调试器，同学们莫搞混淆了哈，它需要一些复杂点的服务端和客户端的配置，而gikdbg.art在正常情况下是不需要手工配置的，所以别去找android server了。对于静态分析，可以执行/ART Debug/View/ELF Data…,/ART Debug/View/ELF Code…两个菜单打开本地so，oat，dex文件。

#### 调试so
```
Step 0.前置说明
手机端：Android模拟器，Android 4.4.2 ART 运行时；（真机与DVM运行时是一样的）
PC端：ParallelDesktop虚拟机，Windows 8.0，gikdbg.art v1.0.build140601.3；
PS:非root环境的设备由于权限的原因会有很多问题，不推荐使用！
```
```
Step 1.连接设备
运行模拟器，打开gikdbg.art.exe，执行/ART Debug/Device菜单，我们就可以来到如下界面：  
```
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_27.png)
```
如果模拟器已经运行了，但是设备列表中没有，则等待一段时间后执行右键的Refresh菜单。然后双击或者右键Login就可以登陆选中的设备了。对于第一次Login该设备，会询问你是否上传依赖的文件到/data/local,这一步如果否定了的话将不能使用调试功能。上传文件这个步骤目前已知的问题是对于非root的设备，往往因为权限的原因上传不成功，一般情况下/data/local/tmp目录没有问题，但是有些设备又没有/data/local/tmp目录，因此我们只有设置/data/local为目标路径，这个问题目前还不知道好的解决办法。归纳一下就是：非root的机器无法在其/data/local下创建我们依赖的文件夹以及上传文件，如果我们将其迁移至/data/local/tmp这个目录下，又有部分设备没有这个文件夹，就更没有办法上传了。
对于这类上传失败的同学，可以想办法手工将$(GIKDBG.ART)/adb/android/gdb传至/data/local/gikir_android-xxxx/gdb这个位置，其中xxxx是GUID。
如果还没有安装该apk文件的，则可以在ADB Shell中执行$install –r命令选择gikdebugee.apk进行安装.
Step 2.选择进程
登陆成功后执行，确保模拟器的gikdebugee.apk运行正常，然后执行/ART Debug/File/Attach就可以得到如下进程列表，选中我们的gikdebugee进程，双击或者执行Attach按钮
```
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_28.png)
```
之后我们就会看到如下加载输出：
```
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_29.png)
```
等gdb加载完毕之后我们就可以进入熟悉的CPU主窗口了：
```
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_30.png)
```
Step 3.选择模块
我们的目的是调试apk里面的so动态库，因此执行/ART Debug/View/Module切换到模块列表，选中我们要调试的模块，双击它
```
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_31.png)
```
Step 4.击中断点
 本例中找到要调试的函数getNativeString，我们可以用CTRL+F查找到它，找到之后F2下断点，F9运行它，然后在设备中操作按钮则该方法将被断点击中，F8运行3步
```
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_32.png)


## 调试Android上Linux程序
```
adb push %NDK%\prebuilt\android-arm\gdbserver\gdbserver /system/bin
chmod 777 /system/bin/gdbserver
adb push test.out /system/bin
chmod 777 /system/bin/test.out
gdbserver :2345 /system/bin/test.out(若附加调试则提供进程号)
adb forward tcp:2345 tcp:2345
gdb >
gdb > target remote :2345
```

### 技巧：如何在so入口下断?
&emsp;&emsp;用ida分析so，并在JNI_OnLoad下断点，动态附加后，ida会自动rebase，使用gdb 的catch load命令捕获

# Java层/Linux层联合调试

## 有源码联合调试
参照前几节

## 无源码联合调试

### 操作步骤
```
adb shell am start -D -n com.example.hellojni/.HelloJni		启动app并等待调试器
	ps | grep hellojni									得到PID 3569
adb shell run-as com.example.hellojni /data/data/com.example.hellojni/lib/gdbserver +debug-socket --attach (3569)PID
	将PID与文件映射建立调试链接(c层)
adb forward tcp:5039 localfilesystem:/data/data/com.example.hellojni/debug-socket将调试链接和本地端口建立链接(c层)
adb forward tcp:65534 jdwp:(3569)PID										 将本地端口和进程建立连接(java层)
jdb -connect com.sun.jdi.SocketAttach:hostname=localhost,port=65534			 使用jdb调试java层
arm-linux-androideabi-gdb.exe											
	target remote :5039												 使用gdb调试c层
	set breakpoint pending on
```

### 最简单的gdb中，断在加载so时刻的方法
* 1.以等待模式启动   
`am start -D -n com.example.hellojni/.HelloJni` 
* 2.Gdbserver链接该进程(ps | grep hello)   
  `gdbserver --attach :1234 10863`
* 3.转发端口    
`adb forward tcp:1234 tcp:1234`
* 4.连接gdb 	  
`arm-linux-androideabi-gdb	Target remote :1234`
```
root@ja3gchnduos:/ # am start -D -n com.example.hellojni/.HelloJni
Starting: Intent { cmp=com.example.hellojni/.HelloJni }
root@ja3gchnduos:/ # ps | grep hello
u0_a165   10863 3593  869292 16088 ffffffff 40077a08 S com.example.hellojni
root@ja3gchnduos:/ # gdbserver --attach :1234 10863
Attached; pid = 10863
Listening on port 1234
```
* 5.设置符号路径(提前把/system/lib/*.so  /system/bin/linker  libhello-jni.so拷贝到目录)  
`set solib-search-path c:/1`
* 6.设置加载so断点  
`catch load libhello-jni.so`
* 7.执行continue，使用android studio的attach使程序继续运行
* 8.加载so时自动断下：
```
Catchpoint 1
  Inferior loaded C:\Users\lichao\sumsing\libhello-jni.so
0x40036b8c in rtld_db_dlactivity () from C:\Users\lichao\sumsing\linker
```
* 9.用ida分析出onload要下断点的偏移，b *addr下断

## Android linux内核层调试

&emsp;&emsp;Android底层为linux层，gdb用于调试linux应用层，而kgdb用于调试linux内核层  
&emsp;&emsp;kgdb的android版本下载：<http://github.com/dankex/kgdb-android>


# 使用Hook

## 常用Hook/Inject工具简介

&emsp;&emsp;常用Hook框架：
* Cydia Substrate    
* * 支持Java层hook
* * 支持Jni层hook
* * 需要Root，且机型适配
* * 支持dalvik，不支持art
* * 闭源
* Xposed <https://github.com/rovo89/Xposed>
* * 支持Java层hook
* * 需要Root，且机型适配
* * 支持dalvik/art
* * 开源
* Frida <https://github.com/frida/frida>
* * 需要Root
* * 支持Java层hook
* * 支持Jni层hook
* * 支持dalvik/art
* * 开源
* * 任意时刻注入，简单易用，远程代码即时编译并注入运行
* Adbi <https://github.com/evilsocket/arminject>
* * 需要Root
* * 支持Jni层hook
* * 任意时刻注入，手工  
* * 开源

# 实例 360手机卫士卸载后弹窗分析过程

## 现象
&emsp;&emsp;360手机卫士在非root情况下卸载后弹出浏览器。于是有2种常见可能，一种是intent跳转，一种是执行am命令，后者可以在java层和jni层实现，如果是java层考虑进行hook，jni层考虑修改am.jar

## 文件注入
&emsp;&emsp;将/system/bin/am改名，发现无法弹窗，于是确定是通过第二种方式实现，为了确定调用层级，尝试修改(反编译成smali->加入logcat输出打印回溯栈和接收参数->回编译)/system/framework/am.jar，(可以通过自己再另一个app中实现同样的功能，通过反编译得到smali代码)再次反编译后内容如下： 
```
 public static void main(String[] args) {
        String v0 = "";
        int v3 = args.length;
        int v2;
        for(v2 = 0; v2 < v3; ++v2) {
            v0 = String.valueOf(v0) + " " + args[v2];
        }
        Log.d("my god", v0);
        Log.d("my god", Log.getStackTraceString(new Throwable()));
    }
```

### 分析日志
&emsp;&emsp;在卸载瞬间拿到输出：  
```
start -n com.android.browser/.BrowserActivity -a android.intent.action.VIEW -d http://shouji.360.cn/web/uninstall/uninstall.html?u=100&id=76bb84de8f53b53f57dd3cedfe966091&v=6.3.1.1048&s=1&model=SE0gTk9URSAxTFRF&sdk=19&ch=200222&wid=9fa298f35aec4232c26048442f36dc59 --user 0
java.lang.Throwable 
	at com.android.commands.am.Am.main(Am.java:30)
	at com.android.internal.os.RuntimeInit.nativeFinishInit(Native Method)
	at com.android.internal.os.RuntimeInit.main(RuntimeInit.java:245)
	at dalvik.system.NativeStart.main(Native Method)
```
发现命令行是 `am start –n com.android.browser/.BrowserActivity -a android.intent.action.VIEW`

### 定位关键代码
&emsp;&emsp;通过字符串搜索，定位到java层关键代码，使用android hook框架cydia substrate，挂钩java.lang.Runtime类的exec函数，定位到调用栈： 
```
content:/data/user/0/com.qihoo360.mobilesafe/files/so_libs/um.0.2 com.qihoo360.mobilesafe --execute am start -n com.android.browser/.BrowserActivity -a android.intent.action.VIEW -d http://shouji.360.cn/web/uninstall/uninstall.html?u=100\&id=7b55c26b779bd111dfed8b02bb00131c\&v=5.5.0.1041\&s=1\&model=TmV4dXMgUw\&sdk=19\&at=KTvooEkhHMzgJ13AXfMkJINnhrmyyNdu\&ch=200222 --user 0
java.lang.Throwable 
	at com.example.emptytest.Main$1$1.invoked(Main.java:68)
	at com.saurik.substrate.MS$2.invoked(MS.java:68)
	at java.lang.Runtime.exec(Native Method)
	at egv.a(360MobileSafe:257)
	at egv.a(360MobileSafe:66)
	at com.qihoo360.mobilesafe.ui.index.MobileSafeApplication.p(360MobileSafe:1223)
	at com.qihoo360.mobilesafe.ui.index.MobileSafeApplication.onCreate(360MobileSafe:799)
```

### 结论
&emsp;&emsp;启动不久，360启动linux程序`/data/data/com.qihoo360.mobilesafe/com.qihoo360.mobilesafe/files/so_libs/um.0.2`，并将弹窗任务以参数形式传递给该程序，程序中对/data/data/com.qihoo360.mobilesafe文件夹的删除操作进行挂钩，以实现卸载后弹窗机制   
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_24.png)

# GDB调试

## 反汇编一段地址
```
(gdb) disass /r 0x401148b8,0x40114900
Dump of assembler code from 0x401148b8 to 0x401148c8:
=> 0x401148b8:  0c 70 a0 e1     mov     r7, r12
   0x401148bc:  01 0a 70 e3     cmn     r0, #4096       ; 0x1000
   0x401148c0:  1e ff 2f 91     bxls    lr
   0x401148c4:  00 00 60 e2     rsb     r0, r0, #0
   0x401148c8:  0e 70 00 ea     b       0x40130908
End of assembler dump.
```

## 表达式计算
```
print expr
	print ”%d” a
    
dprintf 动态插入printf函数
	dprintf location,format string,arg1,arg2,...
```

## 查看寄存器
info registers

## 查看栈参数
info args

## 查看局部变量
info locals

## 查看内存
x

## 修改内存
```
set *(unsigned int*)0x800000000=0x00000000
```

## 断点
break [PROBE_MODIFIER] [LOCATION] [thread THREADNUM] [if CONDITION]
clear [LOCATION]
```
break *0x4000000  绝对地址
break 12          行号
break func1       函数
clear *0x40000000
clear 12
clear func1
```

### 一次断点
tbreak [PROBE_MODIFIER] [LOCATION] [thread THREADNUM] [if CONDITION]

### 条件/线程断点
break func1 thread1 if i==0

### 观察断点
```
watch/awatch/ rwatch [-l|-location] EXPRESSION		变化/读写/读断点
（如果EXPRESSION不是绝对地址，则需要用-l计算表达式）
watch *0x40000000==0x90909090
watch –l *$pc
watch i   (有源码，变量i的值有变化时停止)
```

### 范围断点
```
break-range START-LOCATION, END-LOCATION
break-range 1.c:5, 1.c:10  在1.c的第5行和第10行之间下断
break-range +5, +10	在当前行+5和当前行+10之间下断
```

### 硬件断点
普通硬断 hbreak [PROBE_MODIFIER] [LOCATION] [thread THREADNUM] [if CONDITION]  
临时硬断 thbreak [PROBE_MODIFIER] [LOCATION] [thread THREADNUM] [if CONDITION]  

### 拦截当前函数退出
xbreak

### 捕获断点
普通补断 catch [assert|catch|exception|exec|fork|load|rethrow|signal|syscall|throw|unload|vfork]  
临时捕断 tcatch [assert|catch|exception|exec|fork|load|rethrow|signal|syscall|throw|unload|vfork] 
```
如何使程序在so加载时刻断下??（网上不少人用奇葩方式，还是对gdb不了解）
catch load 1.so
```

### 跟踪断点
strace [LOCATION] [IF CONDITION]

## 查看调用栈
bt [N]		显示N层调用栈  
bt full		显示全部调用栈  

## 流程控制

|行为               |命令      |
|-------------------|---------|
|运行时中断          |Ctrl+C   |
|结束程序            |kill     |
|单步步过            |next     |
|单步步入            |step     |
|单步步过(指令级)     |nexti    |
|单步步入(指令级)     |stepi    |
|继续运行             |continue|
|执行到当前函数指定位置|advance |
|分离进程             |detach  |
|强制跳转             |jump    |

### 反向调试
reverse-continue  reverse-next  reverse-search  reverse-stepi  reverse-finish  reverse-next  reverse-step

## 显示当前加载模块
info shared

## 强制加载模块
可以用于做二进制对比
```
(gdb) load C:/Users/lichao/2/libadnative.so 0x50000000
Loading section .interp, size 0x13 lma 0x50000134
Loading section .dynsym, size 0x1210 lma 0x50000148
Loading section .dynstr, size 0x2061 lma 0x50001358
Loading section .hash, size 0x8a8 lma 0x500033bc
Loading section .rel.dyn, size 0x10a0 lma 0x50003c64
Loading section .rel.plt, size 0x1b0 lma 0x50004d04
Loading section .plt, size 0x29c lma 0x50004eb4
Loading section .text, size 0xe7a0 lma 0x50005150
Loading section .ARM.extab, size 0x8e8 lma 0x500138f0
Loading section .ARM.exidx, size 0xd80 lma 0x500141d8
Loading section .rodata, size 0x11bc lma 0x50014f58
Loading section .data.rel.ro.local, size 0x738 lma 0x50018098
Loading section .fini_array, size 0x8 lma 0x500187d0
Loading section .init_array, size 0x14 lma 0x500187d8
Loading section .data.rel.ro, size 0x508 lma 0x500187f0
Loading section .dynamic, size 0xf8 lma 0x50018cf8
Loading section .got, size 0x210 lma 0x50018df0
Loading section .data, size 0x1c lma 0x50019000
Start address 0x0, load size 94044
Transfer rate: 188 KB/sec, 2541 bytes/write.
```

## 替换当前调试模块
```
file c:/1.so
```

## 进程转储
gcore

## 进程空间
&emsp;&emsp;进程空间inferior，用于调试多个进程，fork函数会自动添加进程空间

|操作         |指令               |
|-------------|------------------|
|添加进程空间  |add-inferior      |
|复制进程空间  |clone-inferior 1  |
|删除进程空间  |remove-inferior 1 |
|切换进程空间  |inferior 2        |
|分离进程空间  |detach inferior 2 |

## 由地址获对应的符号
maintenance translate-address [address]

## 查找符号
```
info functions [regex]   定位地址
info symbol address      定位文件
info variables [regex]   全局静态符号
```

## 执行外部命令
目标系统：! [command]  
主机系统：shell [command]  

## 显示线程
info threads

## 打印c++对象虚表
info vtbl
