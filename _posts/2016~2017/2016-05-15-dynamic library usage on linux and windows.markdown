---
layout: post
title: 对比Linux和Windows关于动态链的使用
categories: Windows
description: 对比linux和windows关于动态链的使用
keywords: dynamic library
---

## Windows情况

&emsp;&emsp;Windows上存在2种方式实现b.dll/b.exe使用a.dll中的函数void func()  
* 编译时确定，使windows在加载和解析(b.dll/b.exe)导入表的时候自动加载a.dll，分2步
* * 添加函数声明：void func();(extern 及 __declspec(dllimport)可选)  
* * 引入导入lib：#pragma comment(lib, "a.lib")  ----***注意导入lib和静态lib的区别***
* 运行时获取，这种方式可以在(b.dll/b.exe)执行的任意时刻加载a.dll并调用函数，也分2步
* * 将模块加载到内存或获取已有模块 HMODULE hmod = LoadLibrary("a.dll",.....)/GetModuleHandle("a.dll")
* * 获取函数并执行 

```C++
typedef void (*F)();F func=(F)GetProcAddress(hmod,"func");func();
```

## Android情况

&emsp;&emsp;Android/Linux上，并不把特定函数归于某个模块，它存在一个模块链，从第一个so开始遍历，直到找到函数为止，同样分2种方式实现b/b.so使用a.so中的函数void func()     下面的方法都是采用ndk
* 编译时确定——这个是我要重点说的，因为网上资料较少，分系统函数和用户函数  
* * 用户函数——自己从网上down的，或自己写的导出函数的so  
Android.mk中编写配置

```Txt
LOCAL_SRC_FILES := a.cpp
LOCAL_MODULE := a
LOCAL_MODULE_FILENAME := a
include $(BUILD_SHARED_LIBRARY)
LOCAL_SRC_FILES := b.cpp
LOCAL_MODULE := b
LOCAL_MODULE_FILENAME := b
LOCAL_SHARED_LIBRARIES := liba      #导入a的函数
include $(BUILD_SHARED_LIBRARY)
```

这样就可以让b去用a的导出函数，b.cpp中仍然要包含a的函数声明

* * 系统函数——ndk自带的库函数，包括stl，log等

Android.mk中编写配置

```Txt
LOCAL_SRC_FILES := b.cpp
LOCAL_MODULE := b
LOCAL_MODULE_FILENAME := b
LOCAL_LDLIBS := -llog      #导入liblog.so中的函数
include $(BUILD_SHARED_LIBRARY)
```

再包含android/log.h即可

系统自带的库参见%NDK%\build\core\build-binary.mk

```Txt
 system_libs := \
     android \
     c \
     dl \
     jnigraphics \
     log \
     m \
     m_hard \
     stdc++ \
     z \
     EGL \
     GLESv1_CM \
     GLESv2 \
     GLESv3 \
     OpenSLES \
     OpenMAXAL \
     bcc \
     bcinfo \
     cutils \
     gui \
     RScpp \
     RScpp_static \
     RS \
     ui \
     utils \
     mediandk \
     atomic
```

* 动态加载，这和windows类似，分2步

```C++
int hmod = dlopen("/system/lib/a.so",
typedef void (*F)();F func=(F)dlsym(hmod,"func");func();
```
