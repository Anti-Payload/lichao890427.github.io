---
layout: post
title: 如何遍历Linux程序的So模块
categories: Android
description: 如何遍历linux程序的so模块
keywords: 
---

# 如何遍历linux程序的so模块

## 代码

```C++
soinfo* si = (soinfo*)dlopen("libdl.so",3);
while(si)
{
    printf("ptr=%08x name=%s entry=%08x base=%08x size=%08x\n",si,si->name,si->entry,si->base,si->size);
    int i;
    for(i=0;i<si->preinit_array_count;i++)
    {
        printf("preinit_array:%08x\n",si->preinit_array[i]);
    }
    for(i=0;i<si->init_array_count;i++)
    {
        printf("init_array:%08x\n",si->init_array[i]);
    }
    for(i=0;i<si->fini_array_count;i++)
    {
        printf("fini_array:%08x\n",si->fini_array[i]);
    }
    printf("init_func:%08x,fini_func:%08x\n",si->init_func,si->fini_func);
    si = si->next;
}
```

## 输出

```Txt
ptr=b6fdc0a8 name=libdl.so entry=00000000 base=00000000 size=00000000
 init_func:00000000,fini_func:00000000
 ptr=b6fc8004 name=./test entry=b6fde704 base=b6fde000 size=00005000
 preinit_array:ffffffff
 preinit_array:00000000
 init_array:ffffffff
 init_array:00000000
 init_array:ffffffff
 init_array:00000000
 fini_array:ffffffff
 fini_array:00000000
 init_func:b6f56140,fini_func:00000000
 ptr=b6fc8128 name=libNimsWrap.so entry=00000000 base=b6fc3000 size=00004000
 init_array:b6fc3881
 fini_array:b6fc3824
 init_func:00000000,fini_func:00000000
 ptr=b6fc824c name=libc.so entry=00000000 base=b6f68000 size=0005b000
 init_array:b6f76471
 init_array:b6f7abc9
 init_array:b6f7abdd
 init_array:b6f7b585
 init_array:b6f7b6bd
 init_array:b6f8d675
 fini_array:b6f771b9
 init_func:00000000,fini_func:00000000
 ptr=b6fc8370 name=libcutils.so entry=00000000 base=b6f5d000 size=0000b000
 fini_array:b6f6038c
 init_func:00000000,fini_func:00000000
 ptr=b6fc8494 name=libAndroidBootstrap0.so entry=00000000 base=b6f55000 size=00008000
 init_array:b6f56ac4
 init_array:00000000
 fini_array:b6f585f4
 fini_array:00000000
 init_func:00000000,fini_func:00000000
 ptr=b6fc85b8 name=libstdc++.so entry=00000000 base=b6f52000 size=00003000
 fini_array:b6f52828
 init_func:00000000,fini_func:00000000
 ptr=b6fc86dc name=libm.so entry=00000000 base=b6f37000 size=0001b000
 fini_array:b6f39940
 init_func:00000000,fini_func:00000000
 ptr=b6fc8800 name=liblog.so entry=00000000 base=b6f12000 size=00005000
 fini_array:b6f12f50
 init_func:00000000,fini_func:00000000
 ptr=b6fc8924 name=libAndroidLoader.so entry=00000000 base=b6f0d000 size=00005000
 init_array:00000000
 fini_array:b6f0e16c
 fini_array:00000000
 init_func:00000000,fini_func:00000000
 ptr=b6fc8a48 name=libsubstrate.so entry=00000000 base=b6f07000 size=00006000
 init_array:00000000
 fini_array:b6f09244
 fini_array:00000000
 init_func:00000000,fini_func:00000000
```

## 结论

&emsp;&emsp;soinfo是个链表结构，从打印的信息来看，是从高地址到低地址排序的，因此要打开一个未加载的so，自然排在高地址位置，因此往后遍历即可