---
layout: post
title: 虚拟机检测程序的分析
categories: Reverse_Engineering
description: 虚拟机检测程序的分析
keywords: 
---

## 背景

&emsp;&emsp;前几个礼拜在网上下了个视频exe(5.exe)，win7下运行直接异常，因此想放到xp虚拟机里看看，结果直接检测出虚拟机，弹窗后直接关闭，很好奇想知道怎么检测的。用windbg加载该exe，运行到弹窗后暂停，“Play请不要在虚拟机中播放！！”。查看调用栈，发现空空如也，感觉不太妙，试着移动窗体，发现居然可以移动！那么这一定是另一个进程的，也就是说5.exe启动了另一个进程。用process explorer查看进程，发现进程树中5.exe居然开启了`C:\WINDOWS\system32\winver.exe`，命令行参数传的是  
`F238FF10FB00F72CD46CD727B402EC259151C620E00AEC26C25EE230F90AEC28C276D135E00CF01D66C260B2C817E732C55E967AF11BE7`  
&emsp;&emsp;之后我在真正的(非虚拟机)win7系统里运行了一下，这个串变成了  
`F238FF01E706F032ED6ECA37FC02ED1DF567D03FE00CF21D842CC62CF1`  
&emsp;&emsp;现在奇怪的事情来了，自己手动执行winver.exe，参数是后面那个长串，预期会弹上面那个窗口，结果却只是winver自己的界面。后来搞了很长时间没有什么进展，无意间我看到5.exe资源(DEL_FD1)里有个pe文件(记为DEL_FD1.exe)，把它dump出来，加上那个参数运行了一下，居然弹窗了，也就说5.exe吧资源在内存dump下来直接运行了，没有通过生成文件这一步。。。而虚拟机检测和参数解析都是DEL_FD1.exe做的，5.exe仅仅用于产生另一个进程。详细看了字符串引用还会有更多发现，原来5.exe用到了BoxedApp SDK来实现资源直接生成进程这一步。如果将资源DEL_FD1替换成其他exe，会发现运行的恰好是替换的这个文件！！！

## 跟踪
&emsp;&emsp;现在目标明确了，用Windbg分析该del_fd1.exe，当然要加上会弹窗的那个参数，再弹窗时，暂停下来看调用栈：

```Txt
0012F67C  77D19418   包含ntdll.KiFastSystemCallRet           USER32.77D19416               0012F6AC
0012F680  00469032  <JMP.&user32.WaitMessage>             DEL_FD1.0046902D              0012F6AC
0012F6B0  004684AC   ? DEL_FD1.00468F00                    DEL_FD1.004684A7              0012F6AC
0012F6D4  00464C28   DEL_FD1.00468490                      DEL_FD1.00464C23              0012F730
```

&emsp;&emsp;这次仍然不是很明显(调Dephi程序好难受，各种奇怪的库函数)，后几个函数跟进去以后可能是窗口回调之类的。没办法，搜字符串吧，0x4a9eb0处可以搜到结果，之后在这里下访问断点，再看调用栈：

```Txt
0012F61C  77D1AE26  ntdll.RtlMultiByteToUnicodeN         USER32.77D1AE20              0012F618
0012F644  77D3C76E   USER32.MBToWCSEx                      USER32.77D3C769               0012F640
0012F674  77D3C730   USER32.DrawTextExA                    USER32.77D3C72B               0012F670
0012F678  01010054     hDC = 01010054
0012F67C  004A9EB0     Text = "请不要在虚拟机中播放!!"
0012F680  00000017     Count = 17 (23.)
0012F684  0012F6E8     pRect = 0012F6E8{0.,0.,960.,0.}
0012F688  00000450     Flags =DT_LEFT|DT_TOP|DT_WORDBREA
0012F68C  00000000     pDrawTextParams =NULL
0012F6A8  0043B898  <JMP.&user32.DrawTextA>               DEL_FD1.0043B893              0012F6A4
0012F6AC  01010054     hDC = 01010054
0012F6B0  004A9EB0     Text = "请不要在虚拟机中播放!!"
0012F6B4  00000017     Count = 17 (23.)
0012F6B8  0012F6E8     pRect = 0012F6E8{0.,0.,960.,0.}
0012F6BC  00000450     Flags =DT_LEFT|DT_TOP|DT_WORDBREA
0012F74C  0043BC52   ? DEL_FD1.0043B68C                    DEL_FD1.0043BC4D              0012F748
0012F758  0043BD9B   ? DEL_FD1.0043BC2C                    DEL_FD1.0043BD96
0012F778  0043BC93   ? DEL_FD1.0043BD34                    DEL_FD1.0043BC8E
0012F77C  00000000     Arg1 = 00000000
0012F780  FFFFFFFF     Arg2 = FFFFFFFF
0012F784  FFFFFFFF     Arg3 = FFFFFFFF
0012F788  00000000     Arg4 = 00000000
0012F794  0043BDCE   DEL_FD1.0043BC74                      DEL_FD1.0043BDC9              0012F790
0012F798  FFFFFFFF     Arg1 = FFFFFFFF
0012F79C  FFFFFFFF     Arg2 = FFFFFFFF
0012F7A0  00000000     Arg3 = 00000000
0012F7A4  0043BDBB   DEL_FD1.0043BDBC                      DEL_FD1.0043BDB6              0012FB78
0012F7A8  004A94F7   DEL_FD1.0043BDB0                      DEL_FD1.004A94F2              0012FB78
```

&emsp;&emsp;可以发现窗口是DrawTextA产生的，怪不得MessageBox断不下呢！输入at 004A94F7转到反汇编，跟到最后终于快得到结果了，可以得到如下关键逻辑：

```C++
if(sub_4A866C() || sub_4A871C() ||sub_4A86CC())
{
	ShowMessage(“请不要在虚拟机中播放”);
	TcustomForm.Close();
}
bool IsUnderVM1()
{
	bool flag=false;
	__try
	{
		_asm
		{
			mov eax,’VMXh’;
			mov ebx,0;
			mov ecx,0Ah;
			mov edx,’VX’;
			in eax,dx;
			cmp ebx,’VMXh’;
			jnz RESULT;
			mov flag,1;
			RESULT:   ;
		}
	}
	__except(1)
	{
		flag=false;
	}
	return flag;
}
bool IsUnderVM2()
{
	bool flag=false;
	__try
	{
		_asm
		{
			mov eax,’VMXh’;
			mov ecx,0Ah;//注意和IsUnderVM1不同
			mov edx,’VX’;
			in eax,dx;
			cmp ebx,’VMXh’
			setz flag;
		}
	}
	__except(1)
	{
		flag=false;
	}
	return flag;
}

/*
.text:004A86CC loc_4A86CC:                             ; CODE XREF:sub_4A8A1C+AEEp
.text:004A86CC                                         ;sub_4A8A1C+B38p
.text:004A86CC                 push    ebp
.text:004A86CD                 mov     ebp, esp
.text:004A86CF                 push    ecx
.text:004A86D0                 push    ebx
.text:004A86D1                 push    esi
.text:004A86D2                 push    edi
.text:004A86D3                 mov     byte ptr [ebp-1], 0
.text:004A86D7                 xor     eax,eax
.text:004A86D9                 push    ebp
.text:004A86DA                 push    offset loc_4A8705//注册异常回调
.text:004A86DF                 push    dword ptr fs:[eax]
.text:004A86E2                 mov     fs:[eax], esp
.text:004A86E5                 push    ebx
.text:004A86E6                 mov     ebx, 0
.text:004A86EB                 mov     eax, 1
.text:004A86EB ;---------------------------------------------------------------------------
.text:004A86F0                 db 0Fh
.text:004A86F1                 db 3Fh
.text:004A86F2                 db 7
.text:004A86F3                 db 0Bh
.text:004A86F4 ;---------------------------------------------------------------------------
.text:004A86F4                 test    ebx, ebx//ZF位为1
.text:004A86F6                 setz    byte ptr [ebp-1]//ebp[-1]=1;
.text:004A86FA                 pop     ebx
.text:004A86FB                 xor     eax, eax
.text:004A86FD                 pop     edx
.text:004A86FE                 pop     ecx
.text:004A86FF                 pop    ecx
.text:004A8700                 mov     fs:[eax], edx
.text:004A8703                 jmp     short loc_4A870F
.text:004A8705 ;---------------------------------------------------------------------------
.text:004A8705
.text:004A8705 loc_4A8705:                             ; DATA XREF:.text:004A86DAo
.text:004A8705                 jmp     @@HandleAnyException ; __linkproc__HandleAnyException
.text:004A870A ;---------------------------------------------------------------------------
.text:004A870A                 call    @@DoneExcept    ; __linkproc__ DoneExcept
.text:004A870F
.text:004A870F loc_4A870F:                             ; CODE XREF:.text:004A8703j
.text:004A870F                 movzx   eax, byte ptr [ebp-1]
.text:004A8713                 pop     edi
.text:004A8714                 pop     esi
.text:004A8715                 pop     ebx
.text:004A8716                 pop     ecx
.text:004A8717                 pop     ebp
.text:004A8718                 retn
*/
```

&emsp;&emsp;中间的0F3F070B是什么，我也不知道于是百度到了这个网站http://hp.vector.co.jp/authors/VA028184/OllyDbgQA.htm，原来是Virtual PC自造的一个指令(不过具体是什么指令我没查到，知道的可以说下)，在其他架构下(如x86)为非法指令会产生异常(情况1)，而Virutal PC下能正常执行(情况2)。下面分别讨论：
* 函数公共部分：ebp[-1]=0,eax=0;004A86D9~ 004A86E2为异常初始化，004A8705为发生异常的回调地址;ebx=0;eax=1
* 情况1：走到004A86F0处，由于不识别指令因此引发异常，直接到004A8705，后面走到004A870F，eax=ebp[-1]=0。
* 情况2：004A86F0处指令正常执行，test指令置ZF位为1，setz的结果是ebp[-1]=1,eax=0,接下来几个pop抵销前面注册异常处理的push，edx刚好对应004A86DF代码处原先入栈的异常帧，movfs:[eax], edx恢复了原始异常链，之后运行到004A870F，eax=ebp[-1]=1。  

原始代码是Delphi写的，这里用MSVC的C代码表示：

```C++
bool IsUnderVM3()
{
	bool flag=false;
	__try
	{
		_asm
		{
			_EMIT 0x0F;
			_EMIT 0x3F;
			_EMIT 0x07;
			_EMIT 0x0B;
		}
		flag=true;
	}
	__except(1)
	{
	}
	return flag;
}
```

## 后记
&emsp;&emsp;现在我对BoxedAppSDK产生了兴趣，以后有时间会研究，感觉很有用，看了一下可以用虚拟方式操作文件、注册表、环境变量、内存、进程。收费蛮高的，从官网上下了一份demo，看了例子有4个：嵌入dll加载，嵌入flash加载，使用内存pe生成进程，使用notepad打开内存文件。这套技术应该是很有用的！！！