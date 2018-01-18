---
layout: post
title: 关于C++名称修饰的解析
categories: C/C++
description: 关于C++名称修饰的解析
keywords: 名称修饰
---

&emsp;&emsp;曾有人发过该功能代码，然而近日发现微软已经提供了专用API实现这一功能，而且比较复杂。该函数为UnDecorateSymbolName，为调试API之一，下面是测试用例：

```C++
#include <DbgHelp.h>  
#include <iostream>  
using namespace std;  
#pragma comment(lib,"Dbghelp.lib")  
   
void main()  
{  
    char str[100];  
    strcpy(str,"?getArgumentTypes@UnDecorator@@CG?AVDName@@XZ");  
    UnDecorateSymbolName(str,str,100,UNDNAME_COMPLETE);  
    cout<<str<<endl;  
} 
```

&emsp;&emsp;为了了解该函数运作原理，我进行资料搜索和部分逆向工作，下面代码便是我的成果:

```C++
#include "undname.h"  
#include <iostream>  
using namespace std;  
   
HANDLE hHeap=NULL;  
   
void * AllocIt( size_t dwBytes)  
{  
    if(dwBytes == 0)  
        return NULL;  
    LPVOID lpMem=HeapAlloc(hHeap,HEAP_ZERO_MEMORY,dwBytes);  
    if(!lpMem)  
        SetLastError(ERROR_NOT_ENOUGH_MEMORY);  
    return lpMem;  
}  
   
void FreeIt(void* lpMem)  
{  
    if(lpMem)  
        HeapFree(hHeap,0,lpMem);  
}  
   
DWORD __stdcall MyUnDecorateSymbolName(  
    PCTSTR DecoratedName,//经过C++修饰的字符串  
    PTSTR UnDecoratedName,//去修饰的字符串  
    DWORD UndecoreatedLength,//修饰字符串长度  
    DWORD Flags)//去修饰标志位  
{  
    int retlen;  
   
    if(DecoratedName == NULL || UnDecoratedName == NULL || UndecoreatedLength < 2)  
    {  
        SetLastError(ERROR_INVALID_PARAMETER);  
        return 0;  
    }  
   
    retlen=0;  
    __try 
    {  
        if(unDName(UnDecoratedName,DecoratedName,UndecoreatedLength-1,AllocIt,FreeIt,(WORD)Flags))  
        {  
            retlen=strlen(UnDecoratedName);  
        }  
    }  
    __except(1)  
    {  
        if(retlen == 0)  
            SetLastError(ERROR_INVALID_PARAMETER);  
    }  
    return retlen;  
}  
   
void main()  
{  
    hHeap=HeapCreate(0,0x100000,0);  
    if(!hHeap)  
        return;  
   
    char str[100];  
    strcpy(str,"?getArgumentTypes@UnDecorator@@CG?AVDName@@XZ");  
    MyUnDecorateSymbolName(str,str,100,UNDNAME_COMPLETE);  
    cout<<str<<endl;  
   
    HeapDestroy(hHeap);  
} 
```