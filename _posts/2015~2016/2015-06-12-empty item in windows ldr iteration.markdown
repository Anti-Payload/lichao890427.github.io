---
layout: post
title: 何遍历Ldr会得到空项？
categories: Windows
description: 何遍历Ldr会得到空项？
keywords: 
---

# 为何遍历Ldr会得到空项？

## 问题

&emsp;&emsp;之前做过ldr遍历的操作，发现第一项竟然是空，也就是大部分元素都是0，下面来揭示一下原理。经过研究，其实Ldr链表得第一项为头结点，为PEB_LDR_DATA结构，而其他所有项均为LDR_DATA_TABLE_ENTRY结构。Ldr的创建位于源码ldrinit.c -> LdrpInitializeProcess

```
 PEB_LDR_DATA PebLdr
 LdrpInitializeProcess   初始化进程时用空项PebLdr创建Ldr
     Peb->Ldr = &PebLdr;
     InitializeListHead(&PebLdr.InLoadOrderModuleList);
     InitializeListHead(&PebLdr.InMemoryOrderModuleList);
     InitializeListHead(&PebLdr.InInitializationOrderModuleList);
     PebLdr.Length = sizeof(PEB_LDR_DATA);
     PebLdr.Initialized = TRUE;
```         
        
&emsp;&emsp;LdrUnloadDll和LdrpLoadDll分别会进行对Ldr这3个链表卸载和增添节点操作，而顺序不同：  
Load时如果发现未加载dll则会增加节点，会先添加InMemoryOrderModuleList/InLoadOrderModuleList两个链表增加节点，之后操作InInitializationOrderModuleList，之后调用DllMain初始化  
而Unload的时候若发现引用计数为0则会删除节点，会先对InMemoryOrderModuleList InInitializationOrderModuleList 两个链表删除节点，之后调用DllMain清理，最后删除InLoadOrderModuleList节点

```C++
#define RemoveEntryList(e) do { PLIST_ENTRY f = (e)->Flink, b = (e)->Blink; f->Blink = b; b->Flink = f; (e)->Flink = (e)->Blink = NULL; } while (0)
```

&emsp;&emsp;可见删除链表操作为将该项后一个节点直接连接到前一个节点，并且将当前节点的首尾指向NULL，因此通过判断Flink=0 可以判断某DLL正在被卸载。正确的遍历Ldr LIST_ENTRY方法：

```C++
    ListHead = &NtCurrentPeb()->Ldr->InLoadOrderModuleList;
    Next = ListHead->Flink;
    while (Next != ListHead)//跳过头结点即可
    {
                  Next = Next->Flink;
    }        
```

而ldr结构图如下：

```C++
typedef struct _PEB_LDR_DATA
 {
     ULONG               Length;
     BOOLEAN             Initialized;
     PVOID               SsHandle;
     LIST_ENTRY          InLoadOrderModuleList;
     LIST_ENTRY          InMemoryOrderModuleList;
     LIST_ENTRY          InInitializationOrderModuleList;
 } PEB_LDR_DATA, *PPEB_LDR_DATA;

 typedef struct _LDR_DATA_TABLE_ENTRY
 {
     LIST_ENTRY InLoadOrderLinks;
     LIST_ENTRY InMemoryOrderModuleList;
     LIST_ENTRY InInitializationOrderModuleList;
     PVOID DllBase;
     PVOID EntryPoint;
     ULONG SizeOfImage;
     UNICODE_STRING FullDllName;
     UNICODE_STRING BaseDllName;
     ULONG Flags;
     USHORT LoadCount;
     USHORT TlsIndex;
     union
     {
         LIST_ENTRY HashLinks;
         struct
         {
             PVOID SectionPointer;
             ULONG CheckSum;
         };
     };
     union
     {
         ULONG TimeDateStamp;
         PVOID LoadedImports;
     };
     PVOID EntryPointActivationContext;
     PVOID PatchInformation;
 } LDR_DATA_TABLE_ENTRY, *PLDR_DATA_TABLE_ENTRY;
```

&emsp;&emsp;当时我遍历的时候将Head当成LDR_DATA_TABLE_ENTRY，自然数据是不对的~~ ，下面是Ldr LIST_ENTRY结构图：  

![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_13.png)