---
layout: post
title: gcc名字修饰解析工具c++filt
categories: C/C++
description: gcc名字修饰解析工具c++filt
keywords: 
---

gcc名字修饰和msvc系列有很大不同，c++filt这个工具可以解析gcc名字修饰，ndk自带工具中包含该程序

## 用法

```Txt
c++filt.exe _ZN7android14AndroidRuntime8startRegEP7_JNIEnv
android::AndroidRuntime::startReg(_JNIEnv*)
c++filt.exe _ZN7android14AndroidRuntime5startEPKcb
android::AndroidRuntime::start(char const*, bool)
c++filt.exe _ZN7android14AndroidRuntime5startEPKcS2_
android::AndroidRuntime::start(char const*, char const*)
c++filt.exe _ZN7android14AndroidRuntime7startVmEPP7_JavaVM
android::AndroidRuntime::startVm(_JavaVM**, _JNIEnv**)
```
