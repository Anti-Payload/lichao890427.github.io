---
layout: post
title: 如何利用宏在调试的时候打出行信息
categories: C/C++
description: 如何利用宏在调试的时候打出行信息
keywords: macro
---

```C++
#define _TOUNICODE(x) L##x  
#define TOUNICODE(x) _TOUNICODE(x)//中介一下，不能直接用  
#define _TOTEXT(x) #x  
#define TOATEXT(x) _TOTEXT(x)//中介一下，不能直接用  
#define TOUTEXT(x) TOUNICODE(TOATEXT(x))  
#if defined(UNICODE) || defined(_UNICODE)  
#define SHOWINFO L"FILE:"##TOUTEXT(__FILE__)##L"\tLINE:"##TOUTEXT(__LINE__)  
#define tcout wcout  
#else  
#define SHOWINFO(x) "FILE:"##TOATEXT(__FILE__)##"\tLINE:"##TOATEXT(__LINE__)  
#define tcout cout  
#endif  

void main()  
{  
    tcout << SHOWINFO << endl;  
    int i;  
    cin >> i;  
}

// 启用调试  
#define NDEBUG  

#ifndef NDEBUG  
using std::cout;  
using std::endl;  
HANDLE coutHandle = GetStdHandle(STD_OUTPUT_HANDLE);  
SetConsoleTextAttribute(coutHandle,FOREGROUND_GREEN);  
cout<< "==============调试模式=================="<<endl  

       << "==============调试模式=================="<<endl;  
SetConsoleTextAttribute(coutHandle,FOREGROUND_INTENSITY);  
#endif

```

