---
layout: post
title: 在Windows驱动编程中使用STL
categories: WindowsDriver
description: 在Windows驱动编程中使用STL
keywords: 
---

## 环境配置
&emsp;&emsp;STL中有很多很好用的数据结构和算法，因此被广泛使用。不用造轮子直接用是懒程序员的直接想法。目前刚学Windows驱动，就想看看能不能直接在驱动中使用STL和BOOST。很多时候这样是不安全的，因为用到了应用层的DLL，可能引起冲突。这里我也是牛刀小试，觉得还是有法可解的。驱动环境安装和VS自动调试环境细节的就不说了：  
* WDK=7600  
* HOSTOS=Win7 x64  
* CLIENTOS=XP Win7 x86  
* VS=2005  
* Debugger=VisualDDK  
&emsp;&emsp;安装好以后，来建立一个空的exe工程testsys，设置vs的include和lib目录  
* include:
* * E:\WinDDK\7600.16385.1\inc\api\crt\stl60  
* * E:\WinDDK\7600.16385.1\inc\ddk  
* * E:\WinDDK\7600.16385.1\inc\crt  
* * E:\WinDDK\7600.16385.1\inc\api  
* lib:  
* * E:\WinDDK\7600.16385.1\lib\win7\i386  
&emsp;&emsp;打开testsys工程属性对话框，如下设置（比网上那些设置要更严格一些）：
```Txt
C/C++
  General:
    Configuration Type:如果能则设为.sys
    Use of MFC:Use MFC in a Static Library
  Optimization:
    Disabled
  Preprocessor:
    WIN32=100;_X86_=1;DBG=1
  Code Generation:
    Enable C++ Exception:Yes
    Basic Runtime Checkefault
    RuntimeLibrary:Multi-threaded Debug(/MTd)
    Buffer Security Check:No
  Precompiled Headers:Not Using..
  Advanced
    Calling Convention:_stdcall
Linker
  General
    Output Fle:后缀改sys
    Enable Incremental Linking:No
  Input
    Additional Dependencies:ntoskrnl
    Ignore All Default Libraries:Yes
  Manifest File:No
  System
    SubSystem:Native
    Heap Reserve Size:40000
    Heap Commit Size:1000
    Driverriver
  Advanced
    Entry PointriverEntry
    Base Address:0x10000
    Target Machine:MachineX86
```

## 编写测试用例
testdriver.cpp

```C++
#include <ntddk.h>  
#include "ntmem.h"  
#include <string>  
#include <vector>  
#include <numeric>  
#include <map>  
using namespace std;  
::_Lockit::_Lockit(){};  
::_Lockit::~_Lockit(){};  
void _cdecl std::_Xlen(void){};  
void _cdecl std::_Xran(void){};  
extern "C" NTSTATUS DriverEntry(IN PDRIVER_OBJECT pDriverObject,IN PUNICODE_STRING)  
{  
    //如何保证用stl等c++库不会出问题，我觉得有下面几点：  
    //1 不使用带有io操作的，尽量使用内存操作的  
    //2 使用静态链接，不要使用导入库，那样会引入应用层dll  
    vector<int> vec;  
    for(int i=1;i<=100;i++)  
    {  
        vec.push_back(i);  
    }  
    int result1 = accumulate(vec.begin(),vec.end(),0);  
    KdPrint(("1-100 sum=%d\n",result1));  
   
    string result2="lichao";  
    result2 += "890427";  
    KdPrint(("name=%s\n",result2.c_str()));  
       
    map<string,string> result3;  
    result3["telphone"]="18392635617";  
    KdPrint(("tel=%s\n",result3["telphone"].c_str()));  
   
    NTSTATUS status = STATUS_SUCCESS;  
    return status;  
} 
```

```C++
ntmem.h
#pragma once  
#ifndef _LCNTDEFS_  
#define _LCNTDEFS_  
   
#ifdef new  
#undef new  
#endif  
#ifdef delete  
#undef delete  
#endif  
   
//common new  
void* __cdecl operator new(size_t _Size)  
{  
    return ExAllocatePool(NonPagedPool,_Size);  
}  
   
//array new  
void* _cdecl operator new[](size_t _Size)  
{  
    return ExAllocatePool(NonPagedPool,_Size);  
}  
   
//common delete  
inline void _cdecl operator delete(void* _P)  
{  
    if(_P)  
        ExFreePool(_P);  
}  
   
//array delete  
inline void _cdecl operator delete[](void* _P)  
{  
    if(_P)  
        ExFreePool(_P);  
}  
   
#endif/*_LCNTDEFS_*/  
   
extern "C"  
{  
    void _cdecl __CxxFrameHandler3()  
    {  
   
    }  
    int _cdecl memcmp(const void* buf1,const void* buf2,size_t count)  
    {  
        return RtlCompareMemory(buf1,buf2,count);  
    }  
    int __security_cookie=0;  
} 
```

编译结果：
```Txt
1>------ Build started: Project: testsys, Configuration: Debug Win32 ------
1>Compiling...
1>cl : Command line warning D9007 : '/Gm' requires '/Zi or /ZI'; option ignored
1>cl : Command line warning D9002 : ignoring unknown option '/verbose'
1>testdriver.cpp
1>c:\users\lichao\desktop\testdriver\testsys\ntmem.h(48) : warning C4244: 'return' : conversion from 'SIZE_T' to 'int', possible loss of data
1>Linking...
1>Starting pass 1
1>LINK : warning LNK4075: ignoring '/DELAYLOAD' due to '/DRIVER' specification
1>Searching libraries
1>    Searching E:\WinDDK\7600.16385.1\lib\win7\i386\ntoskrnl.lib:
1>      Found __imp__ExAllocatePool@8
1>        Referenced in testdriver.obj
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found __imp__ExFreePoolWithTag@8
1>        Referenced in testdriver.obj
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found __imp__RtlCompareMemory@12
1>        Referenced in testdriver.obj
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found _DbgPrint
1>        Referenced in testdriver.obj
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found _strlen
1>        Referenced in testdriver.obj
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found _memcpy
1>        Referenced in testdriver.obj
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found _memmove
1>        Referenced in testdriver.obj
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found __IMPORT_DESCRIPTOR_ntoskrnl
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found __NULL_IMPORT_DESCRIPTOR
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>      Found ntoskrnl_NULL_THUNK_DATA
1>        Referenced in ntoskrnl.lib(ntoskrnl.exe)
1>        Loaded ntoskrnl.lib(ntoskrnl.exe)
1>    Searching E:\WinDDK\7600.16385.1\lib\Crt\i386\\DelayImp.lib:
1>Finished searching libraries
1>Finished pass 1
1>Searching libraries
1>    Searching E:\WinDDK\7600.16385.1\lib\win7\i386\ntoskrnl.lib:
1>      Found __load_config_used
1>        Loaded ntoskrnl.lib(loadcfg.obj)
1>    Searching E:\WinDDK\7600.16385.1\lib\Crt\i386\\DelayImp.lib:
1>Finished searching libraries
1>Starting pass 2
1>     testdriver.obj
1>     ntoskrnl.lib(loadcfg.obj)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>     ntoskrnl.lib(ntoskrnl.exe)
1>Finished pass 2
1>roject : warning PRJ0018 : The following environment variables were not found:
1>$(WindowsSdkDir)
1>Build log was saved at "file://c:\Users\lichao\Desktop\testdriver\testsys\Debug\BuildLog.htm"
1>testsys - 0 error(s), 4 warning(s)
========== Build: 1 succeeded, 0 failed, 0 up-to-date, 0 skipped ==========
```
&emsp;&emsp;注意，ntmem.h重载了new和delete操作符，并且处理了unresolved symbols错误（__CxxFrameHandler3，memcmp，__security_cookie），如果今后有找不到链接符号的时候，在头文件里写相应的实现即可  
&emsp;&emsp;输出结果：
```Txt
1-100 sum=5050
name=l
tel=18392635617
```

## 总结
&emsp;&emsp;此次测试，对string类，vector类及map类做了简单测试，未发生蓝屏，说明其实驱动里，还是可以用stl的，因为stl基本上是内存操作。
