---
layout: post
title: 如何捕获程序mov eax,fs:[0]的行为?
categories: Debug
description: 如何捕获程序mov eax,fs:[0]的行为?
keywords: 
---

&emsp;&emsp;这个问题我今天学习调试时突发奇想想到的，FS:[0]处能做不少事情，反调试、获取tib peb数据、异常处理，如果能提前拦截到程序mov eax,fs:[0]，mov eax,fs:[0x30]等行为会不会是个很好的想法呢？以下是我的研究成果：

```C++
void main()
{
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm mov eax,FS:[0];
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm mov eax,FS:[0x30];
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
    _asm nop;
}
```

&emsp;&emsp;编译以后，使用调试器打开exe。对于ollydbg，为了精确控制代码范围，打开查看->源文件,选择1.cpp后，查看其源代码，并右键在第一个nop和最后一个nop处下断点。fs对应线性地址直接在寄存器一栏给出，我这里是7EFDD000，因此下硬件断点：hr 7EFDD000，如果设置成功，则调试->硬件断点就能看到，测试结束后从这里删除。对于windbg，使用命令dg fs得到fs:[0]对应线性地址，我的结果如下：

|    |        |        |          |P Si Gr Pr Lo|        |
|----|--------|--------|----------|-------------|--------|
|Sel |  Base  |  Limit |   Type   |l ze an es ng|Flags   |
|0053|7efdd000|00000fff|Data RW Ac|3 Bg By P  Nl|000004f3|  

&emsp;&emsp;接着下4字节硬件访问断点 ba r4 7efdd000，分别运行程序，会发现在访问fs:[0]处断下！fs:[0x30]可以同理类推，硬件断点和内存断点十分有用，尤其在破解和动态调试时，例如破解试用期，只需暂停程序，搜索内存，找到之后下访问断点，就能找到执行分支，快速定位关键代码！！

