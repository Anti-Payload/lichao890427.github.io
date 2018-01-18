---
layout: post
title: 函数指针和函数引用
categories: C/C++
description: 函数指针和函数引用
keywords: 
---

## 简介

&emsp;&emsp;无论任何书都会有这段内容：
指针和引用是双胞胎，前者是C而后者是C++的，两者底层实现完全一样(曾有一本很nb的书让我放弃这个看法)，事实上也不可能不一样，引用的话必须绑定当前已存在的值或对象，定义时就要以该值或对象初始化，之后的修改均看做对原值或对象的操作。函数指针你是知道的，他就是指向函数的指针，那函数引用呢？现在我提出函数引用这个定义，定义为必须用函数初始化的函数变量来对比一下，看此例：

```C++
#include "stdio.h"
//声明类型
typedef int (&MYGETCHAR1)(void);//函数指针形式1
typedef int (*MYGETCHAR2)(void);//函数引用形式2
int main(int argc, char* argv[])
{
	MYGETCHAR1 c=getchar;
	MYGETCHAR2 d=getchar;
	//定义变量
	int (*func0)(void)=getchar;//函数指针形式2
	int (&fund0)(void)=getchar;//函数引用形式2
	int (*func1)(void)=NULL;//正确
	int (&fund1)(void)=NULL;//编译错误
	int (*func2)(void)=NULL;//正确
	int (&fund2)(void)=NULL;//编译错误
	return 0;
}
```

&emsp;&emsp;代码中MYGETCHAR1是函数引用而MYGETCHAR2是函数指针，和函数指针一样，引用指针也有2种形式。同样，函数引用也必须初始化
