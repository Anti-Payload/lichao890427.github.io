---
layout: post
title: Swig-Python-C++高级用法
categories: Python
description: Swig-Python-C++高级用法
keywords:
---

# Swig-Python-C++高级用法
Swig为多种语言之间建立通信机制，实现相互调用，这篇文章说说Swig实现Python调用C++模块的方式(http://www.swig.org/survey.html)。  
以下测试代码均使用环境：Win7_x64 + Python2.7(x86) + Swig + Mingw开发，由于编译Python用到gcc因此使用Windows上的Mingw由于Mingw使用x86因此python选择x86

## 简单案例
```
/* File: example.i */
%module example
%{
#define SWIG_FILE_WITH_INIT
%}

%inline%{
	int fact(int n) {
		if (n < 0){ /* This should probably return an error, but this is simpler */
		return 0;
		}
		if (n == 0) {
		return 1;
		}
		else {
		/* testing for overflow would be a good idea here */
		return n * fact(n-1);
		}
	}
%}
```

* 产生相关文件  
执行`swig -c++ -python example.i`后会产生example.py,example_wrap.cxx两个文件，py封装了python层基本调用，而cxx封装了c++的python接口，需要编译成可执行代码由example.py调用。  
* 编译cxx生成_example.pyd
`gcc -O2 -fPIC example_wrap.cxx -IC:\Python27_86\include -lpython27 -LC:/Python27_86/libs  -lstdc++ -shared -o _example.pyd`
* 调用  
```
C:\download>c:\python27_86\python.exe
Python 2.7.14 (v2.7.14:84471935ed, Sep 16 2017, 20:19:30) [MSC v.1500 32 bit (Intel)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>> import example
>>> example.fact(11)
39916800
```

编译64位：  
编译64位模块比较繁琐，需要所有相关的可执行模块都是64位，包括python本身，linux中可以增加-m32/-m64选项指定平台位

