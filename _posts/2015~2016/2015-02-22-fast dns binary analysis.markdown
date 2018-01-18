---
layout: post
title: 对彗星DNS加速器的分析
categories: Reverse_Engineering
description: 对彗星DNS加速器的分析
keywords: dns
---

# 对彗星DNS加速器的分析

## 背景介绍
&emsp;&emsp;FASTDNS作者是张慧星，该软件能很好地测试dns并自动设置系统，一直以来都在用却不知道原理，这2天我做了逆向分析，并打算将delphi的移植为vc++的源码，如果有可能以后会兼容ipv6 dns。  
&emsp;&emsp;提前声明：手动标注是逆向分析的重要步骤，下面代码中有些是API名，其余大部分有名称的都是分析了函数以后手动做的命名，为分析结构方便，以下汇集了我3天的成果，目的就是为了用VC++完成同样的功能，并期盼以后可以改进，比如兼容Ipv6和结合代理软件。  

## 开始分析
&emsp;&emsp;看到设置界面发现可以设置多线程的，因此从这里入手，bp CreateThread，发现断不下来，于是bm *!CreateThread*，断到这里：

```Asm
ecx=param0
edx=dwStackSize
eax=lpThreadAttributes
param1=lpThreadId
param2=dwCreationFlags
param3 
0043D348 push    ebp 
0043D349 mov     ebp, esp 
0043D34B push    ebx 
0043D34C push    esi 
0043D34D push    edi 
0043D34E mov     edi, ecx//edi=param1  
0043D350 mov     esi, edx//esi=dwStackSize  
0043D352 mov     ebx, eax//ebx=lpThreadAttributes  
0043D354 mov     eax, 8//DWORD[2]  
0043D359 call    sub_43B5C0  
0043D35E mov     [eax], edi//DWORD[0]=param0;FARPROC  
0043D360 mov     edx, [ebp+arg_8]  
0043D363 mov     [eax+4], edx//DWORD[1]=param3;PARAM  
0043D366 mov     byte_4B9C2D, 1  
0043D36D mov     edx, [ebp+dwCreationFlags_1]  
0043D370 push    edx                            ; lpThreadId  
0043D371 mov     edx, [ebp+dwCreationFlags]  
0043D374 push    edx                            ; dwCreationFlags  
0043D375 push    eax                            ; lpParameter  
0043D376 mov     eax, offset SpeedTest  
0043D37B push    eax                            ; lpStartAddress  
0043D37C push    esi                            ; dwStackSize  
0043D37D push    ebx                            ; lpThreadAttributes  
0043D37E call    CreateThread  
0043D383 pop     edi 
0043D384 pop     esi 
0043D385 pop     ebx 
0043D386 pop     ebp 
0043D387 retn    0Ch 
```

&emsp;&emsp;可以看到实际只执行了参数中的函数指针部分，现在我们来找CreateThread的StartAddress和Param部分，往上层找到StartAddress的值为Classes::_17197  
```Asm
004254A4 @Classes@TThread@$bctr$qqro proc near  ; CODE XREF: sub_40E908+33 p  
04254E6 push    esi                            ; param3  
004254E7 push    4                              ; dwCreationFlags  
004254E9 lea     eax, [esi+8]  
004254EC push    eax                            ; lpThreadId  
004254ED mov     ecx, offset @Classes@_17197    ; a3  
004254F2 xor     edx, edx                       ; dwStackSize  
004254F4 xor     eax, eax                       ; lpThreadAttributes  
004254F6 call    @BeginThread 

00425404 @Classes@_17197 proc near              ; DATA XREF: Classes::TThread::TThread(bool)+49 o  
00425404  
00425404 var_4= dword ptr -4  
00425404  
00425404 push    ebp 
00425405 mov     ebp, esp 
00425407 push    ecx 
00425408 push    ebx 
00425409 push    esi 
0042540A push    edi 
0042540B mov     [ebp+var_4], eax 
0042540E xor     eax, eax 
00425410 push    ebp 
00425411 push    offset j_@@HandleFinally_0     ; __linkproc__ HandleFinally  
00425416 push    dword ptr fs:[eax]  
00425419 mov     fs:[eax], esp 
0042541C mov     eax, [ebp+var_4]  
0042541F cmp     byte ptr [eax+0Dh], 0  
00425423 jnz     short loc_42545A  
00425425 xor     eax, eax 
00425427 push    ebp 
00425428 push    offset loc_425445  
0042542D push    dword ptr fs:[eax]  
00425430 mov     fs:[eax], esp 
00425433 mov     eax, [ebp+var_4]  
00425436 mov     edx, [eax]  
00425438 call    dword ptr [edx+4]  
0042543B xor     eax, eax 
0042543D pop     edx 
0042543E pop     ecx 
0042543F pop     ecx 
00425440 mov     fs:[eax], edx 
00425443 jmp     short loc_42545A  
00425445 ; ---------------------------------------------------------------------------  
00425445  
00425445 loc_425445:                            ; DATA XREF: Classes::_17197+24 o  
00425445 jmp     @@HandleAnyException           ; __linkproc__ HandleAnyException  
0042544A ; ---------------------------------------------------------------------------  
0042544A call    @AcquireExceptionObject  
0042544F mov     edx, [ebp+var_4]  
00425452 mov     [edx+2Ch], eax 
00425455 call    @@DoneExcept                   ; __linkproc__ DoneExcept  
0042545A  
0042545A loc_42545A:                            ; CODE XREF: Classes::_17197+1F j  
0042545A                                        ; Classes::_17197+3F j  
0042545A xor     eax, eax 
0042545C pop     edx 
0042545D pop     ecx 
0042545E pop     ecx 
0042545F mov     fs:[eax], edx 
00425462 push    offset unk_42549C  
00425467  
00425467 loc_425467:                            ; CODE XREF: .text:0042549A j  
00425467 mov     eax, [ebp+var_4]  
0042546A mov     bl, [eax+0Fh]  
0042546D mov     eax, [ebp+var_4]  
00425470 mov     esi, [eax+14h]  
00425473 mov     eax, [ebp+var_4]  
00425476 mov     byte ptr [eax+10h], 1  
0042547A mov     eax, [ebp+var_4]  
0042547D mov     edx, [eax]  
0042547F call    dword ptr [edx]  
00425481 test    bl, bl 
00425483 jz      short loc_42548D  
00425485 mov     eax, [ebp+var_4]  
00425488 call    @TObject@Free                  ; TObject::Free  
0042548D  
0042548D loc_42548D:                            ; CODE XREF: Classes::_17197+7F j  
0042548D mov     eax, esi                       ; dwExitCode  
0042548F call    @EndThread 
```

&emsp;&emsp;实际执行的又是参数中的函数指针；先执行*(param+4)(param) 在执行*(param(param),（感觉delphi的这个结构构造的很巧妙，传入CreateThread的param之前在堆上分配，而进了startaddress后自动释放，param里面也是StartAddress+Param的结构^_^可以无限这么套下去），而param的值需要再往上找。  
```Asm
004254A4 @Classes@TThread@$bctr$qqro proc near  ; CODE XREF: sub_40E908+33 p  
004254A4  
004254A4 var_10= dword ptr -10h  
004254A4 var_C= dword ptr -0Ch  
004254A4 var_8= byte ptr -8  
004254A4 var_1= byte ptr -1  
004254A4  
004254A4 push    ebp 
004254A5 mov     ebp, esp 
004254A7 add     esp, 0FFFFFFF0h  
004254AA push    ebx 
004254AB push    esi 
004254AC xor     ebx, ebx 
004254AE mov     [ebp+var_10], ebx 
004254B1 test    dl, dl 
004254B3 jz      short loc_4254BD  
004254B5 add     esp, 0FFFFFFF0h  
004254B8 call    @@ClassCreate                  ; __linkproc__ ClassCreate  
004254BD  
004254BD loc_4254BD:                            ; CODE XREF: Classes::TThread::TThread(bool)+F j  
004254BD mov     ebx, ecx 
004254BF mov     [ebp+var_1], dl 
004254C2 mov     esi, eax 
004254C4 xor     eax, eax 
004254C6 push    ebp 
004254C7 push    offset loc_42554E  
004254CC push    dword ptr fs:[eax]             ; lpThreadIda  
004254CF mov     fs:[eax], esp 
004254D2 xor     edx, edx 
004254D4 mov     eax, esi 
004254D6 call    @TObject@$bctr_0               ; TObject::`...'  
004254DB call    sub_42522C  
004254E0 mov     [esi+0Eh], bl 
004254E3 mov     [esi+0Ch], bl 
004254E6 push    esi                            ; param3  
004254E7 push    4                              ; dwCreationFlags  
004254E9 lea     eax, [esi+8]  
004254EC push    eax                            ; lpThreadId  
004254ED mov     ecx, offset @Classes@_17197    ; a3  
004254F2 xor     edx, edx                       ; dwStackSize  
004254F4 xor     eax, eax                       ; lpThreadAttributes  
004254F6 call    @BeginThread 
```

&emsp;&emsp;看到004254E6处push的第一个参数位esi，往上看到004254C2处eax做了赋值，往前再没给eax赋值的，因此eax是传入参数，还要往调用者找，（delphi程序喜欢用ecx eax edx传值，不过这样的话在调用系统api前后，要做栈<-->寄存器参数变换）  
```Asm
0040E908 sub_40E908 proc near                   ; CODE XREF: sub_4094B4+46 p  
0040E908 push    ebp 
0040E909 mov     ebp, esp 
0040E90B add     esp, 0FFFFFFD4h  
0040E90E mov     [ebp+var_8], dl 
0040E911 test    dl, dl 
0040E913 jle     short loc_40E91A  
0040E915 call    __ClassCreate  
0040E91A  
0040E91A loc_40E91A:                            ; CODE XREF: sub_40E908+B j  
0040E91A mov     [ebp+var_2A], cl 
0040E91D mov     [ebp+var_29], dl 
0040E920 mov     [ebp+var_4], eax 
0040E923 mov     eax, offset unk_4B3C0C  
0040E928 call    @__InitExceptBlockLDTC  
0040E92D mov     [ebp+var_18], 8  
0040E933 mov     cl, [ebp+var_2A]  
0040E936 xor     edx, edx 
0040E938 mov     eax, [ebp+var_4]  
0040E93B call    @Classes@TThread@$bctr$qqro    ; Classes::TThread::TThread(bool) 
```

&emsp;&emsp;看到0040E938处var_4对eax赋值，而往上0040E920处eax又赋值给var_4，之前eax再没有赋值行为，同样eax也是该函数参数，继续往调用者走  
```Asm
004094B4 push    ebp 
004094B5 mov     ebp, esp 
004094B7 add     esp, 0FFFFFFD4h  
004094BA push    ebx 
004094BB push    esi 
004094BC push    edi 
004094BD mov     [ebp+var_8], dl 
004094C0 test    dl, dl 
004094C2 jle     short loc_4094C9  
004094C4 call    __ClassCreate  
004094C9  
004094C9 loc_4094C9:                            ; CODE XREF: sub_4094B4+E j  
004094C9 mov     ebx, ecx 
004094CB mov     [ebp+var_29], dl 
004094CE mov     [ebp+var_4], eax 
004094D1 mov     eax, offset unk_4B24D0  
004094D6 call    @__InitExceptBlockLDTC  
004094DB mov     [ebp+var_18], 8  
004094E1 mov     edx, [ebp+arg_10]  
004094E4 mov     ecx, ebx 
004094E6 push    edx 
004094E7 push    [ebp+arg_C]  
004094EA push    [ebp+arg_8]  
004094ED mov     eax, [ebp+arg_4]  
004094F0 push    eax 
004094F1 mov     edx, [ebp+arg_0]  
004094F4 push    edx 
004094F5 xor     edx, edx 
004094F7 mov     eax, [ebp+var_4]  
004094FA call    sub_40E908 

00402748 push    ebp 
00402749 mov     ebp, esp 
0040274B add     esp, 0FFFFFF98h  
0040274E mov     eax, offset unk_4A2608  
00402753 push    ebx 
00402754 push    esi 
00402755 push    edi 
00402756 mov     esi, [ebp+arg_0]  
00402759 call    @__InitExceptBlockLDTC  
0040275E mov     ebx, [ebp+arg_4]  
00402761 cmp     byte ptr [ebx+10h], 1  
00402765 jnz     loc_4028A0  
0040276B mov     [ebp+var_34], 8  
00402771 push    ebx 
00402772 lea     eax, [ebp+var_4C]  
00402775 mov     [ebp+var_4C], offset sub_402990  
0040277C mov     [ebp+var_48], esi 
0040277F mov     dl, 1  
00402781 push    dword ptr [eax+4]  
00402784 push    dword ptr [eax]  
00402786 mov     ecx, [esi+3ECh]  
0040278C push    ecx 
0040278D mov     cl, 1  
0040278F mov     eax, [esi+3F0h]  
00402795 push    eax 
00402796 mov     eax, off_4B2618  
0040279B call    sub_4094B4 
```

&emsp;&emsp;最后找到这里，发现eax是4B2618，来看该处2个函数指针(应该说是虚函数表，从后面可以看到param其实是个类)：
```Asm
.data:004B2618 off_4B2618 dd offset _cls_DNSSpending_TDNSSpending//虚表
。。。。。。。。。。。//类成员

.data:004B2664 _cls_DNSSpending_TDNSSpending dd offset @TThread@DoTerminate
.data:004B2664                                         ; DATA XREF: .text:00409906 o
.data:004B2664                                         ; .dataff_4B2618 o
.data:004B2664                                         ; TThread:oTerminate
.data:004B2668 dd offset sub_40EA3C
.data:004B266C dd offset sub_40970C
```

&emsp;&emsp;再往上走可以看到程序按照分的线程数创建线程：   
```Asm
00402940 sub_402940      proc near              ; CODE XREF: showstatus+43 p  
00402940                                        ; _TfrmFastDNS_actTestSpendingExecute+138 p  
00402940  
00402940 var_8           = dword ptr -8  
00402940 var_4           = dword ptr -4  
00402940 arg_0           = dword ptr  8  
00402940 arg_4           = dword ptr  0Ch  
00402940  
00402940                 push    ebp 
00402941                 mov     ebp, esp 
00402943                 add     esp, 0FFFFFFF8h  
00402946                 push    ebx 
00402947                 push    esi 
00402948                 push    edi 
00402949                 mov     edi, [ebp+arg_4]  
0040294C                 mov     ebx, [ebp+arg_0]  
0040294F                 cmp     dword ptr [ebx+400h], 0  
00402956                 jz      short loc_402988  
00402958                 xor     esi, esi 
0040295A                 cmp     edi, esi 
0040295C                 jle     short loc_402988  
0040295E  
0040295E loc_40295E:                            ; CODE XREF: sub_402940+46 j  
0040295E                 push    0  
00402960                 push    0  
00402962                 mov     [ebp+var_8], offset sub_402748  
00402969                 mov     [ebp+var_4], ebx 
0040296C                 lea     eax, [ebp+var_8]  
0040296F                 push    dword ptr [eax+4]  
00402972                 push    dword ptr [eax]  
00402974                 mov     ecx, [ebx+400h]  
0040297A                 push    ecx 
0040297B                 call    sub_40C400  
00402980                 add     esp, 14h  
00402983                 inc     esi 
00402984                 cmp     edi, esi 
00402986                 jg      short loc_40295E  
00402988  
00402988 loc_402988:                            ; CODE XREF: sub_402940+16 j  
00402988                                        ; sub_402940+1C j  
00402988                 pop     edi 
00402989                 pop     esi 
0040298A                 pop     ebx 
0040298B                 pop     ecx 
0040298C                 pop     ecx 
0040298D                 pop     ebp 
0040298E                 retn 
0040298E sub_402940      endp 
```

&emsp;&emsp;再往上走就是TfrmFastDNS_actTestSpendingExecute，也就是点击了测试按钮之后的回调函数  
```Asm
004033D4 _TfrmFastDNS_actTestSpendingExecute proc near 
004033D4                                        ; DATA XREF: .data:fastdns___CPPdebugHook+13E3 o  
004033D4  
004033D4 var_38          = dword ptr -38h  
004033D4 var_34          = dword ptr -34h  
004033D4 var_30          = dword ptr -30h  
004033D4 var_20          = word ptr -20h  
004033D4 var_14          = dword ptr -14h  
004033D4 var_C           = byte ptr -0Ch  
004033D4 var_8           = byte ptr -8  
004033D4 var_4           = byte ptr -4  
004033D4  
004033D4                 push    ebp 
004033D5                 mov     ebp, esp 
004033D7                 add     esp, 0FFFFFFC8h  
004033DA                 push    ebx 
004033DB                 mov     ebx, eax 
004033DD                 mov     eax, offset unk_4A2954  
004033E2                 call    @__InitExceptBlockLDTC  
004033E7                 xor     edx, edx 
004033E9                 mov     eax, [ebx+304h]  
004033EF                 call    sub_443028  
004033F4                 xor     edx, edx 
004033F6                 mov     eax, [ebx+364h]  
004033FC                 call    sub_443028  
00403401                 mov     byte ptr [ebx+404h], 0  
00403408                 call    @GetCurrentTime  
0040340D                 mov     [ebx+3E8h], eax 
00403413                 mov     edx, 8000000Fh  
00403418                 mov     eax, [ebx+350h]  
0040341E                 call    @TControl@SetColor ; TControl::SetColor  
00403423                 mov     [ebp+var_20], 8  
00403429                 mov     edx, offset unk_4A22AA  
0040342E                 lea     eax, [ebp+var_4]  
00403431                 call    @System@AnsiString@$bctr$qqrpxc ; System::AnsiString::AnsiString(char *)  
00403436                 inc     [ebp+var_14]  
00403439                 mov     edx, [eax]  
0040343B                 mov     eax, [ebx+350h]  
00403441                 call    @TControl@SetText ; TControl::SetText  
00403446                 dec     [ebp+var_14]  
00403449                 lea     eax, [ebp+var_4]  
0040344C                 mov     edx, 2  
00403451                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
00403456                 mov     dl, 1  
00403458                 mov     eax, [ebx+35Ch]  
0040345E                 call    unknown_libname_1860 ; Delphi2006/BDS2006 Visual Component Library  
0040345E                                        ; BDS 4.0 RTL and VCL  
00403463                 mov     [ebp+var_20], 14h  
00403469                 mov     edx, offset unk_4A22AB  
0040346E                 lea     eax, [ebp+var_8]  
00403471                 call    @System@AnsiString@$bctr$qqrpxc ; System::AnsiString::AnsiString(char *)  
00403476                 inc     [ebp+var_14]  
00403479                 mov     edx, [eax]  
0040347B                 mov     eax, [ebx+34Ch]  
00403481                 call    @TControl@SetText ; TControl::SetText  
00403486                 dec     [ebp+var_14]  
00403489                 lea     eax, [ebp+var_8]  
0040348C                 mov     edx, 2  
00403491                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
00403496                 xor     edx, edx 
00403498                 mov     eax, [ebx+30Ch]  
0040349E                 call    sub_443028  
004034A3                 mov     eax, [ebx+400h]  
004034A9                 test    eax, eax 
004034AB                 jz      short loc_4034CC  
004034AD                 push    0  
004034AF                 push    0  
004034B1                 mov     [ebp+var_38], offset sub_4033B0  
004034B8                 mov     [ebp+var_34], ebx 
004034BB                 lea     edx, [ebp+var_38]  
004034BE                 push    dword ptr [edx+4]  
004034C1                 push    dword ptr [edx]  
004034C3                 push    eax 
004034C4                 call    sub_40C400  
004034C9                 add     esp, 14h  
004034CC  
004034CC loc_4034CC:                            ; CODE XREF: _TfrmFastDNS_actTestSpendingExecute+D7 j  
004034CC                 xor     eax, eax 
004034CE                 xor     edx, edx 
004034D0                 mov     [ebx+37Ch], eax 
004034D6                 lea     eax, [ebp+var_C]  
004034D9                 mov     [ebp+var_20], 20h  
004034DF                 call    unknown_libname_1523 ; CBuilder 4 and Delphi 4 VCL  
004034DF                                        ; CBuilder 5 runtime  
004034E4                 inc     [ebp+var_14]  
004034E7                 mov     edx, [eax]  
004034E9                 mov     eax, [ebx+358h]  
004034EF                 call    @TControl@SetText ; TControl::SetText  
004034F4                 dec     [ebp+var_14]  
004034F7                 lea     eax, [ebp+var_C]  
004034FA                 mov     edx, 2  
004034FF                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
00403504                 mov     ecx, [ebx+3F4h]  
0040350A                 push    ecx 
0040350B                 push    ebx 
0040350C                 call    sub_402940  
00403511                 add     esp, 8  
00403514                 mov     eax, [ebp+var_30]  
00403517                 mov     large fs:0, eax 
0040351D                 pop     ebx 
0040351E                 mov     esp, ebp 
00403520                 pop     ebp 
00403521                 retn 
00403521 _TfrmFastDNS_actTestSpendingExecute endp 
00403521 
```

可知以下3个函数为关键函数，下断点：
* bp 425690 =>STOP
* bp 40ea3c =>FUNC1
* bp 40970c =>FUNC2

&emsp;&emsp;发现正如上面分析那样，调用顺序为FUNC1 -> FUNC2 -> STOP  FUNC2通过调用堆栈可以得知是在FUNC1中调用的。先来分析三个函数类型：前面425404已经得知第一个和第二个，先断在425690，看调用栈，通过在上层函数中查看当前函数调用之前的状况粗略分析类型：
```Txt
0:001> k
ChildEBP RetAddr  
WARNING: Stack unwind information not available. Following frames may be wrong.
0273ff74 0043d33a FastDNS!Rtflabelinitialization$qqrv+0xd7d8
0273ff88 76b2919f FastDNS!Rtflabelinitialization$qqrv+0x25482
0273ff94 7745b5af KERNEL32!BaseThreadInitThunk+0xe
0273ffdc 7745b57a ntdll!__RtlUserThreadStart+0x2f
0273ffec 00000000 ntdll!_RtlUserThreadStart+0x1b
```

```Asm
0043D311 mov     ebp, esp 
0043D313 call    @System@_16542                 ; System::_16542  
0043D318 push    ebp 
0043D319 xor     ecx, ecx 
0043D31B push    offset @System@_16754          ; System::_16754  
0043D320 mov     edx, fs:[ecx]  
0043D323 push    edx 
0043D324 mov     fs:[ecx], esp 
0043D327 mov     eax, [ebp+arg_0]  
0043D32A mov     ecx, [eax+4]  
0043D32D mov     edx, [eax]  
0043D32F push    ecx 
0043D330 push    edx 
0043D331 call    sub_43B5E0  
0043D336 pop     edx 
0043D337 pop     eax 
0043D338 call    edx 
0043D33A xor     edx, edx 
0043D33C pop     ecx 
0043D33D mov     fs:[edx], ecx 
0043D340 pop     ecx 
0043D341 pop     ebp 
0043D342 pop     ebp 
0043D343 retn    4 
```

&emsp;&emsp;根据栈平衡，同时call edx处可能传递的参数应该在当前call之前，上个call之后，加上调用者代码可以推断使用了eax，没有用到栈参数，其他2函数同理，再到STOP中看：（显然这是终止线程之类的收尾工作，所以这里我取名STOP）
```Asm
.text:00425690 ; TThread:oTerminate
.text:00425690 @TThread@DoTerminate proc near          ; DATA XREF: .text:_cls_Classes_TThread o
.text:00425690                                         ; .data:_cls_CalcSpendingThread_TCalcSpendingThread o
.text:00425690 cmp     word ptr [eax+1Ah], 0
.text:00425695 jz      short locret_4256A2
.text:00425697 push    eax
.text:00425698 push    offset @Controls@TSizeConstraints@Change$qqrv ; Delphi 5 Visual Component Library
.text:00425698                                         ; Delphi2006/BDS2006 Visual Component Library
.text:0042569D call    terminate
.text:004256A2
.text:004256A2 locret_4256A2:                          ; CODE XREF: TThread:oTerminate+5 j
.text:004256A2 retn

0040EA3C sub_40EA3C      proc near              ; DATA XREF: .data:fastdns___CPPdebugHook+105D0 o  
0040EA3C                                        ; .data:fastdns___CPPdebugHook+11C3C o  
0040EA3C  
0040EA3C var_14          = dword ptr -14h  
0040EA3C var_10          = dword ptr -10h  
0040EA3C pertime         = dword ptr -0Ch  
0040EA3C var_8           = dword ptr -8  
0040EA3C a1              = dword ptr -4  
0040EA3C  
0040EA3C                 push    ebp 
0040EA3D                 mov     ebp, esp 
0040EA3F                 add     esp, 0FFFFFFECh  
0040EA42                 mov     [ebp+a1], eax 
0040EA45                 mov     eax, [ebp+a1]  
0040EA48                 mov     byte ptr [eax+0Fh], 1  
0040EA4C                 lea     edx, [ebp+pertime]  
0040EA4F                 push    edx 
0040EA50                 mov     ecx, [ebp+a1]  
0040EA53                 push    dword ptr [ecx+30h]  
0040EA56                 push    [ebp+a1]  
0040EA59                 call    sub_40EAE0  
0040EA5E                 add     esp, 0Ch  
0040EA61                 test    al, al 
0040EA63                 jz      short loc_40EA8B  
0040EA65                 push    [ebp+var_8]  
0040EA68                 push    [ebp+pertime]  
0040EA6B                 mov     edx, [ebp+a1]  
0040EA6E                 push    dword ptr [edx+30h]  
0040EA71                 call    sub_40EABC  
0040EA76                 add     esp, 0Ch  
0040EA79                 push    0  
0040EA7B                 mov     ecx, [ebp+a1]  
0040EA7E                 push    dword ptr [ecx+30h]  
0040EA81                 call    sub_40EAD0  
0040EA86                 add     esp, 8  
0040EA89                 jmp     short loc_40EA9B  
0040EA8B ; ---------------------------------------------------------------------------  
0040EA8B  
0040EA8B loc_40EA8B:                            ; CODE XREF: sub_40EA3C+27 j  
0040EA8B                 push    3  
0040EA8D                 mov     eax, [ebp+a1]  
0040EA90                 push    dword ptr [eax+30h]  
0040EA93                 call    sub_40EAD0  
0040EA98                 add     esp, 8  
0040EA9B  
0040EA9B loc_40EA9B:                            ; CODE XREF: sub_40EA3C+4D j  
0040EA9B                 mov     [ebp+var_14], offset sub_40EA10  
0040EAA2                 mov     edx, [ebp+a1]  
0040EAA5                 mov     [ebp+var_10], edx 
0040EAA8                 lea     ecx, [ebp+var_14]  
0040EAAB                 push    dword ptr [ecx+4] ; a5  
0040EAAE                 push    dword ptr [ecx] ; a4  
0040EAB0                 mov     eax, [ebp+a1]  ; a1  
0040EAB3                 call    terminate  
0040EAB8                 mov     esp, ebp 
0040EABA                 pop     ebp 
0040EABB                 retn 

0040EAE0 sub_40EAE0      proc near              ; CODE XREF: sub_40EA3C+1D p  
0040EAE0  
0040EAE0 var_40          = qword ptr -40h  
0040EAE0 var_38          = dword ptr -38h  
0040EAE0 usedtime        = qword ptr -34h  
0040EAE0 totoaltime      = dword ptr -2Ch  
0040EAE0 var_28          = dword ptr -28h  
0040EAE0 var_18          = word ptr -18h  
0040EAE0 var_C           = dword ptr -0Ch  
0040EAE0 var_4           = byte ptr -4  
0040EAE0 arg_0           = dword ptr  8  
0040EAE0 arg_4           = dword ptr  0Ch  
0040EAE0 arg_8           = dword ptr  10h  
0040EAE0  
0040EAE0                 push    ebp 
0040EAE1                 mov     ebp, esp 
0040EAE3                 add     esp, 0FFFFFFC0h  
0040EAE6                 mov     eax, offset unk_4B3C64  
0040EAEB                 call    @__InitExceptBlockLDTC  
0040EAF0                 mov     edx, [ebp+arg_0]  
0040EAF3                 push    dword ptr [edx+40h]  
0040EAF6                 push    [ebp+arg_4]  
0040EAF9                 call    unknown_libname_1142 ; CBuilder 5 runtime  
0040EAFE                 add     esp, 8  
0040EB01                 mov     [ebp+var_18], 8  
0040EB07                 lea     eax, [ebp+var_4]  
0040EB0A                 call    unknown_libname_1128 ; CBuilder 5 runtime  
0040EB0F                 inc     [ebp+var_C]  
0040EB12                 mov     [ebp+var_18], 14h  
0040EB18                 xor     edx, edx 
0040EB1A                 mov     [ebp+totoaltime], edx 
0040EB1D                 xor     ecx, ecx 
0040EB1F                 mov     dword ptr [ebp+usedtime+4], ecx 
0040EB22                 jmp     short loc_40EB61  
0040EB24 ; ---------------------------------------------------------------------------  
0040EB24  
0040EB24 loc_40EB24:                            ; CODE XREF: sub_40EAE0+8A j  
0040EB24                 lea     eax, [ebp+usedtime]  
0040EB27                 push    eax 
0040EB28                 mov     edx, [ebp+arg_0]  
0040EB2B                 push    edx 
0040EB2C                 mov     ecx, [edx]  
0040EB2E                 call    dword ptr [ecx+8]  
0040EB31                 add     esp, 8  
0040EB34                 test    al, al 
0040EB36                 jz      short loc_40EB54  
0040EB38                 mov     eax, dword ptr [ebp+usedtime]  
0040EB3B                 add     [ebp+totoaltime], eax 
0040EB3E                 push    [ebp+arg_4]  
0040EB41                 call    @AspHlpr@TMTSASPObject@getObjectContext$qv_4 ; CBuilder 5 runtime  
0040EB41                                        ; BCC v4.x/5.x class library 32 bit  
0040EB46                 pop     ecx 
0040EB47                 dec     eax 
0040EB48                 push    eax 
0040EB49                 push    [ebp+arg_4]  
0040EB4C                 call    unknown_libname_1142 ; CBuilder 5 runtime  
0040EB51                 add     esp, 8  
0040EB54  
0040EB54 loc_40EB54:                            ; CODE XREF: sub_40EAE0+56 j  
0040EB54                 mov     eax, 64h       ; dwMilliseconds  
0040EB59                 call    @Sleep  
0040EB5E                 inc     dword ptr [ebp+usedtime+4]  
0040EB61  
0040EB61 loc_40EB61:                            ; CODE XREF: sub_40EAE0+42 j  
0040EB61                 mov     edx, dword ptr [ebp+usedtime+4]  
0040EB64                 mov     ecx, [ebp+arg_0]  
0040EB67                 cmp     edx, [ecx+40h]  
0040EB6A                 jl      short loc_40EB24  
0040EB6C                 push    [ebp+arg_4]  
0040EB6F                 call    @AspHlpr@TMTSASPObject@getObjectContext$qv_4 ; CBuilder 5 runtime  
0040EB6F                                        ; BCC v4.x/5.x class library 32 bit  
0040EB74                 pop     ecx 
0040EB75                 mov     edx, [ebp+arg_0]  
0040EB78                 cmp     eax, [edx+40h]  
0040EB7B                 jz      short loc_40EBC9  
0040EB7D                 push    [ebp+arg_4]  
0040EB80                 call    @AspHlpr@TMTSASPObject@getObjectContext$qv_4 ; CBuilder 5 runtime  
0040EB80                                        ; BCC v4.x/5.x class library 32 bit  
0040EB85                 pop     ecx 
0040EB86                 mov     ecx, [ebp+arg_0]  
0040EB89                 mov     edx, [ecx+40h]  
0040EB8C                 sub     edx, eax 
0040EB8E                 mov     [ebp+var_38], edx 
0040EB91                 fild    [ebp+var_38]  
0040EB94                 mov     eax, [ebp+totoaltime]  
0040EB97                 mov     dword ptr [ebp+var_40], eax 
0040EB9A                 xor     ecx, ecx 
0040EB9C                 mov     dword ptr [ebp+var_40+4], ecx 
0040EB9F                 fild    [ebp+var_40]  
0040EBA2                 fdivrp  st(1), st 
0040EBA4                 mov     eax, [ebp+arg_8]  
0040EBA7                 fstp    qword ptr [eax]  
0040EBA9                 mov     al, 1  
0040EBAB                 push    eax 
0040EBAC                 dec     [ebp+var_C]  
0040EBAF                 lea     eax, [ebp+var_4]  
0040EBB2                 mov     edx, 2  
0040EBB7                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
0040EBBC                 pop     eax 
0040EBBD                 mov     edx, [ebp+var_28]  
0040EBC0                 mov     large fs:0, edx 
0040EBC7                 jmp     short loc_40EBE7  
0040EBC9 ; ---------------------------------------------------------------------------  
0040EBC9  
0040EBC9 loc_40EBC9:                            ; CODE XREF: sub_40EAE0+9B j  
0040EBC9                 xor     eax, eax 
0040EBCB                 push    eax 
0040EBCC                 dec     [ebp+var_C]  
0040EBCF                 lea     eax, [ebp+var_4]  
0040EBD2                 mov     edx, 2  
0040EBD7                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
0040EBDC                 pop     eax 
0040EBDD                 mov     edx, [ebp+var_28]  
0040EBE0                 mov     large fs:0, edx 
0040EBE7  
0040EBE7 loc_40EBE7:                            ; CODE XREF: sub_40EAE0+E7 j  
0040EBE7                 mov     esp, ebp 
0040EBE9                 pop     ebp 
0040EBEA                 retn 
0040EBEA sub_40EAE0      endp 
```

&emsp;&emsp;可以得知该函数循环执行指定次数的dns连接，使用的时间累加到totaltime中，后面使用该时间计算平均用时，整个程序精华在于  
` (unsigned __int8)(*(int (__cdecl **)(int, int *))(*(_DWORD *)a1 + 8))(a1, &usedtime)`  
&emsp;&emsp;这个call，实际上是FUNC2，而sub_40EA3C中可以解释为：
```Asm
0040EA3C sub_40EA3C      proc near              ; DATA XREF: .data:fastdns___CPPdebugHook+105D0 o  
0040EA3C                                        ; .data:fastdns___CPPdebugHook+11C3C o  
0040EA3C  
0040EA3C var_14          = dword ptr -14h  
0040EA3C var_10          = dword ptr -10h  
0040EA3C pertime         = dword ptr -0Ch  
0040EA3C var_8           = dword ptr -8  
0040EA3C a1              = dword ptr -4  
0040EA3C  
0040EA3C                 push    ebp 
0040EA3D                 mov     ebp, esp 
0040EA3F                 add     esp, 0FFFFFFECh  
0040EA42                 mov     [ebp+a1], eax 
0040EA45                 mov     eax, [ebp+a1]  
0040EA48                 mov     byte ptr [eax+0Fh], 1  
0040EA4C                 lea     edx, [ebp+pertime]  
0040EA4F                 push    edx 
0040EA50                 mov     ecx, [ebp+a1]  
0040EA53                 push    dword ptr [ecx+30h]  
0040EA56                 push    [ebp+a1]  
0040EA59                 call    sub_40EAE0  
0040EA5E                 add     esp, 0Ch  
0040EA61                 test    al, al 
0040EA63                 jz      short loc_40EA8B  
0040EA65                 push    [ebp+var_8]  
0040EA68                 push    [ebp+pertime]  
0040EA6B                 mov     edx, [ebp+a1]  
0040EA6E                 push    dword ptr [edx+30h]  
0040EA71                 call    sub_40EABC  
0040EA76                 add     esp, 0Ch  
0040EA79                 push    0  
0040EA7B                 mov     ecx, [ebp+a1]  
0040EA7E                 push    dword ptr [ecx+30h]  
0040EA81                 call    sub_40EAD0  
0040EA86                 add     esp, 8  
0040EA89                 jmp     short loc_40EA9B  
0040EA8B ; ---------------------------------------------------------------------------  
0040EA8B  
0040EA8B loc_40EA8B:                            ; CODE XREF: sub_40EA3C+27 j  
0040EA8B                 push    3  
0040EA8D                 mov     eax, [ebp+a1]  
0040EA90                 push    dword ptr [eax+30h]  
0040EA93                 call    sub_40EAD0  
0040EA98                 add     esp, 8  
0040EA9B  
0040EA9B loc_40EA9B:                            ; CODE XREF: sub_40EA3C+4D j  
0040EA9B                 mov     [ebp+var_14], offset sub_40EA10  
0040EAA2                 mov     edx, [ebp+a1]  
0040EAA5                 mov     [ebp+var_10], edx 
0040EAA8                 lea     ecx, [ebp+var_14]  
0040EAAB                 push    dword ptr [ecx+4] ; a5  
0040EAAE                 push    dword ptr [ecx] ; a4  
0040EAB0                 mov     eax, [ebp+a1]  ; a1  
0040EAB3                 call    terminate  
0040EAB8                 mov     esp, ebp 
0040EABA                 pop     ebp 
0040EABB                 retn 
```

&emsp;&emsp;现在同时推测param类结构，可知
```Txt
+00h virtualtable
          +0 STOP
          +4 FUNC1
          +8 FUNC2
+04h HANDLE hthread ??
+0Fh bool Initaled ??
+10h 
+18h FARPROC func0
+1Ch LPVOID param0
+20h FARPROC func1//OnIdle处理函数
+24h LPVOID param1
+28h BOOL IsMainthread??
+30h struct*
                +00h int leftTime//剩余次数
                +08h double pertime//平均消耗时间
                +10h flag statu  //成功0 失败3
+38h FARPROC func2
+3Ch DWORD param2
+40h int repeatTime//取样次数
+44h int timeout//超时
```

## 得到设置DNS逻辑
&emsp;&emsp;通过字符串搜索，可以定位到TfrmSetDNS_btnConfirmClick函数为点击了设置dns窗口的“设置”按钮后回调函数  
```C++
__int32 *__fastcall TfrmSetDNS_btnConfirmClick(int a1, int a2)  
{  
  v32 = a2;  
  v33 = a1;  
  __InitExceptBlockLDTC();  
  v31 = &v48;  
  v52 = 0;  
  v53 = 0;  
  v54 = 0;  
  v36 += 4;  
  v35 = 8;  
  for ( i = 0; i < (*(int (**)(void))(**(_DWORD **)(*(_DWORD *)(v33 + 772) + 536) + 20))(); ++i )// 0044a604 GetCount 选中DNS地址个数  
  {  
    v35 = 44;  
    v3 = *(_DWORD *)(v33 + 0x304);  
    addrstring = 0;  
    ++v36;  
    (*(void (__fastcall **)(_DWORD, int, int *))(**(_DWORD **)(v3 + 0x218)  
                                               + 12))(// 0044a620  getaddrstring  获取地址文本串  
      *(_DWORD *)(v3 + 0x218),  
      i,  
      &addrstring);  
    if ( v53 == v54 )  
    {  
      sub_403C34((int)&v52, v53, (int)&addrstring, (int)&v30, 1u, 1);  
    }  
    else 
    {  
      v46 = v53;  
      if ( v53 )  
      {  
        v35 = 56;  
        System::AnsiString::AnsiString(v46, &addrstring);  
        v35 = 44;  
      }  
      v53 += 4;  
    }  
    --v36;  
    System::AnsiString::~AnsiString(&addrstring, 2);  
  }  
  ++v36;  
  v29 = &v45;  
  v49 = 0;  
  v50 = 0;  
  ++v36;  
  v51 = 0;  
  ++v36;  
  ++v36;  
  ++v36;  
  --v36;  
  v35 = 8;  
  for ( j = 0; j < (*(int (**)(void))(**(_DWORD **)(*(_DWORD *)(v33 + 768) + 536) + 20))(); ++j )// 对于网卡列表中的每一项  
  {  
    if ( (unsigned __int8)TCheckListBox::GetChecked(*(_DWORD *)(v33 + 768), j) )// 如果复选框选中  
    {  
      v35 = 92;  
      v5 = *(_DWORD *)(v33 + 768);  
      v44 = 0;  
      ++v36;  
      (*(void (__fastcall **)(_DWORD, int, int *))(**(_DWORD **)(v5 + 536) + 12))(*(_DWORD *)(v5 + 536), j, &v44);// 获取网卡名称  
      if ( v50 == v51 )  
      {  
        sub_403C34((int)&v49, v50, (int)&v44, (int)&v28, 1u, 1);  
      }  
      else 
      {  
        v43 = v50;  
        if ( v50 )  
        {  
          v35 = 104;  
          System::AnsiString::AnsiString(v43, &v44);  
          v35 = 92;  
        }  
        v50 += 4;  
      }  
      --v36;  
      System::AnsiString::~AnsiString(&v44, 2);  
    }  
  }  
  if ( (unsigned int)((v53 - v52) / 4) <= 50 )  // 如果选中dns地址小于50个  
  {  
    if ( (v50 - v49) / 4 )                      // 如果选中了网卡  
    {  
      v35 = 284;  
      v10 = sub_407DCC((int)off_4A4404, 1, v33);  
      sub_407F6C(v10, (int)&v49, (int)&v52);      
      (*(void (**)(void))(*(_DWORD *)v10 + 0xE8))();// 004670dc  showmodel and setsetting 设置dns并显示进度  
      *(_BYTE *)(v33 + 776) = (unsigned int)*(_BYTE *)(v10 + 776) < 1;  
      v35 = 8;  
      v37 = v10;  
      if ( v10 )  
      {  
        v38 = *(_DWORD *)v10;  
        v35 = 308;  
        (*(void (__fastcall **)(int, signed int))(*(_DWORD *)v37 - 4))(v37, 3);  
        v35 = 296;  
      }  
      sub_466E84(v33);  
      --v36;  
      sub_403254(v49, v50);  
      v35 = 8;  
      --v36;  
      v9 = (v51 - v49) / 4;  
      if ( v49 )  
      {  
        v8 = 4 * v9;  
        if ( (unsigned int)(4 * v9) <= 0x80 )  
          sub_4A0CA4(v49, v8);  
        else 
          _rtl_close(v49);  
      }  
      --v36;  
      --v36;  
      --v36;  
      sub_403254(v52, v53);  
      --v36;  
      result = (__int32 *)v52;  
      v7 = (v54 - v52) / 4;  
      if ( v52 )  
      {  
        if ( (unsigned int)(4 * v7) <= 0x80 )  
          result = sub_4A0CA4(v52, 4 * v7);  
        else 
          result = (__int32 *)_rtl_close(v52);  
      }  
    }  
    else 
    {  
      v35 = 200;  
      ++v36;  
      FormatString(&v39, (char *)"请至少选择一个连接。");  
      v36 += 4;  
      MessageBox((int)&v39);  
      --v36;  
      v35 = 8;  
      --v36;  
      v19 = v40 - v39;  
      if ( v39 )  
      {  
        if ( (unsigned int)v19 <= 0x80 )  
          sub_4A0CA4(v39, v19);  
        else 
          _rtl_close(v39);  
      }  
      --v36;  
      --v36;  
      --v36;  
      (*(void (**)(void))(**(_DWORD **)(v33 + 768) + 192))();  
      --v36;  
      v18 = v50;  
      v17 = v49;  
      sub_403254(v49, v50);  
      --v36;  
      v16 = (v51 - v49) / 4;  
      if ( v49 )  
      {  
        v15 = 4 * v16;  
        if ( (unsigned int)(4 * v16) <= 0x80 )  
          sub_4A0CA4(v49, v15);  
        else 
          _rtl_close(v49);  
      }  
      --v36;  
      --v36;  
      --v36;  
      v14 = v53;  
      v13 = v52;  
      sub_403254(v52, v53);  
      --v36;  
      v12 = (v54 - v52) / 4;  
      if ( v52 )  
      {  
        v11 = 4 * v12;  
        if ( (unsigned int)(4 * v12) <= 0x80 )  
          sub_4A0CA4(v52, v11);  
        else 
          _rtl_close(v52);  
      }  
      --v36;  
      --v36;  
      result = v34;  
    }  
  }  
  else 
  {  
    v35 = 116;  
    ++v36;  
    FormatString(&handle, "选中的DNS地址数量过多。\n只允许选择不超过50个的DNS地址。");  
    v36 += 4;  
    MessageBox((int)&handle);  
    --v36;  
    v35 = 8;  
    --v36;  
    v27 = v42 - handle;  
    if ( handle )  
    {  
      if ( (unsigned int)v27 <= 0x80 )  
        sub_4A0CA4(handle, v27);  
      else 
        _rtl_close(handle);  
    }  
    --v36;  
    --v36;  
    --v36;  
    (*(void (**)(void))(**(_DWORD **)(v33 + 764) + 192))();  
    --v36;  
    v26 = v50;  
    v25 = v49;  
    sub_403254(v49, v50);  
    --v36;  
    v24 = (v51 - v49) / 4;  
    if ( v49 )  
    {  
      if ( (unsigned int)(4 * v24) <= 0x80 )  
        sub_4A0CA4(v49, 4 * v24);  
      else 
        _rtl_close(v49);  
    }  
    --v36;  
    --v36;  
    --v36;  
    v23 = v53;  
    v22 = v52;  
    result = (__int32 *)sub_403254(v52, v53);  
    --v36;  
    v21 = (v54 - v52) / 4;  
    if ( v52 )  
    {  
      v20 = 4 * v21;  
      if ( (unsigned int)(4 * v21) <= 0x80 )  
        result = sub_4A0CA4(v52, v20);  
      else 
        result = (__int32 *)_rtl_close(v52);  
    }  
    --v36;  
    --v36;  
  }  
  return result;  
} 
``` 

&emsp;&emsp;`(*(void (**)(void))(*(_DWORD *)v10 + 0xE8))()`通过消息机制最终调用了`_TfrmSetDNSProccess_tmrStartTimer`，该函数真正执行设置dns：  
对于每个网卡接口  
&emsp;&emsp;00407F88先清除dns   
&emsp;&emsp;&emsp;&emsp;netsh interface ip set dns name="??网"  source=static addr=none  
&emsp;&emsp;对于每个dns地址  
&emsp;&emsp;&emsp;&emsp;004082F4逐个添加dns  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;netsh interface ip add dns name="??网" addr=??.??.??.??   

```C++
int __fastcall TfrmSetDNSProccess_tmrStartTimer(int a1)  
{  
  int v1; // ebx@1  
  int i; // edi@1  
  int v3; // ST0C_4@2  
  int *v4; // eax@2  
  int v5; // esi@4  
  int v6; // ST0C_4@5  
  int v7; // ST08_4@5  
  int *v8; // eax@5  
  int v10; // [sp+Ch] [bp-30h]@4  
  int v11; // [sp+34h] [bp-8h]@5  
  int v12; // [sp+38h] [bp-4h]@2  
   
  v1 = a1;  
  __InitExceptBlockLDTC();  
  unknown_libname_1860(*(_DWORD *)(v1 + 760), 0);  
  sub_486B60(*(_DWORD *)(v1 + 752), 1);  
  sub_486B6C(  
    *(_DWORD *)(v1 + 752),  
    ((*(_DWORD *)(*(_DWORD *)(v1 + 768) + 4) - **(_DWORD **)(v1 + 768)) / 4 + 1)  
  * ((*(_DWORD *)(*(_DWORD *)(v1 + 764) + 4) - **(_DWORD **)(v1 + 764))  
   / 4));  
  for ( i = **(_DWORD **)(v1 + 764); i != *(_DWORD *)(*(_DWORD *)(v1 + 764) + 4); i += 4 )  
  {  
    v3 = *(_DWORD *)i;  
    v12 = 0;  
    v4 = (int *)System::AnsiString::sprintf(&v12, (const char *)&unk_4A40BB, v3);  
    TControl::SetText(*(_DWORD *)(v1 + 756), *v4);// 显示清除原有设置  
    System::AnsiString::~AnsiString(&v12, 2);  
    TApplication::ProcessMessages(*off_4B9450[0]);  
    if ( !cleardns(v1, i) )  
    {  
      *(_BYTE *)(v1 + 776) = 1;  
      return sub_466E84(v1);  
    }  
    TProgressBar::StepBy(*(_DWORD *)(v1 + 752), 1);// 更新进度条  
    v5 = **(_DWORD **)(v1 + 768);  
    v10 = 1;  
    while ( v5 != *(_DWORD *)(*(_DWORD *)(v1 + 768) + 4) )  
    {  
      v6 = *(_DWORD *)v5;  
      v7 = *(_DWORD *)i;  
      v11 = 0;  
      v8 = (int *)System::AnsiString::sprintf(&v11, (const char *)&unk_4A40D9, v7, v6);  
      TControl::SetText(*(_DWORD *)(v1 + 756), *v8);// 显示正在写入dns  
      System::AnsiString::~AnsiString(&v11, 2);  
      TApplication::ProcessMessages(*off_4B9450[0]);  
      if ( adddns(v1, i, v5, v10) )  
        ++v10;  
      TProgressBar::StepBy(*(_DWORD *)(v1 + 752), 1);  
      v5 += 4;  
    }  
  }  
  return sub_466E84(v1);  
} 

bool __cdecl cleardns(int a1, int a2)  
{  
  __InitExceptBlockLDTC();  
  qmemcpy(&v17, &unk_4A3F18, 0x40u);  
  do 
  {  
    v15 = 2;  
    ExitCode = -1;  
    v10 = *(_DWORD *)(a1 + 772);  
    v18 = 8;  
    ++v19;  
    v9 = *(_DWORD *)a2;  
    v32 = 0;  
    ++v19;  
    sub_40CA68((int)&v32, (int)&v17, 64);  
    if ( v32 )  
      v2 = v32;  
    else 
      v2 = (const char *)&unk_4A3F8E;  
    v31 = 0;  
    ++v19;  
    v14 = (char **)System::AnsiString::sprintf(&v31, v2, v9);// 构造netsh命令  
    if ( *v14 )  
      v3 = *v14;  
    else 
      v3 = (char *)&unk_4A3F8F;  
    FormatString(&v29, v3);  
    v19 += 4;  
    ++v19;  
    v28 = 0;  
    ++v19;  
    unknown_libname_1857(*off_4B9450[0], &v28);  
    s = 0;  
    ++v19;  
    ExtractFilePath(v28, &s);  
    if ( s )  
      v4 = s;  
    else 
      v4 = (char *)&unk_4A3F8D;  
    FormatString(&handle, v4);  
    v19 += 4;  
    v11 = (unsigned __int8)sub_40E30C((int)&handle, (int)&v29, v10, &ExitCode) < 1u;// CreateProcess   netsh程序  
    --v19;  
    --v19;  
    v13 = v26 - handle;  
    if ( handle )  
    {  
      if ( (unsigned int)v13 <= 0x80 )  
        sub_4A0CA4(handle, v13);  
      else 
        _rtl_close(handle);  
    }  
    --v19;  
    --v19;  
    --v19;  
    --v19;  
    System::AnsiString::~AnsiString(&s, 2);  
    --v19;  
    System::AnsiString::~AnsiString(&v28, 2);  
    --v19;  
    --v19;  
    v12 = v30 - v29;  
    if ( v29 )  
    {  
      if ( (unsigned int)v12 <= 0x80 )  
        sub_4A0CA4(v29, v12);  
      else 
        _rtl_close(v29);  
    }  
    --v19;  
    --v19;  
    --v19;  
    --v19;  
    System::AnsiString::~AnsiString(&v32, 2);  
    --v19;  
    System::AnsiString::~AnsiString(&v31, 2);  
    if ( v11 )  
    {  
      v18 = 68;  
      v5 = System::AnsiString::AnsiString(&v24, &unk_4A3F90);// 调用的动态库不存在。  
      ++v19;  
      (*(void (__fastcall **)(_DWORD, _DWORD))(**(_DWORD **)(a1 + 772) + 44))(*(_DWORD *)(a1 + 772), *(_DWORD *)v5);  
      --v19;  
      System::AnsiString::~AnsiString(&v24, 2);  
      ExitCode = 9999;  
    }  
    if ( ExitCode )  
    {  
      v18 = 80;  
      ++v19;  
      v23 = 0;                                  // 设置中遇到问题。设置停止。.若不能解决问题，测试结果仍然有效，可据此在网络属性中手工设置DNS。  
      ++v19;  
      (*(void (__fastcall **)(_DWORD, int *))(**(_DWORD **)(a1 + 772) + 28))(*(_DWORD *)(a1 + 772), &v23);  
      v22 = 0;  
      ++v19;  
      v6 = (char **)System::AnsiString::sprintf(&v22, (const char *)&unk_4A3FA5, v23);  
      if ( *v6 )  
        v7 = *v6;  
      else 
        v7 = (char *)&unk_4A40AA;  
      FormatString(&v20, v7);  
      v19 += 4;  
      v15 = sub_40E26C((int)&v20, 64, 5, 0, 0);  
      --v19;  
      --v19;  
      if ( v20 )  
      {  
        if ( (unsigned int)(v21 - v20) <= 0x80 )  
          sub_4A0CA4(v20, v21 - v20);  
        else 
          _rtl_close(v20);  
      }  
      --v19;  
      --v19;  
      --v19;  
      --v19;  
      System::AnsiString::~AnsiString(&v23, 2);  
      --v19;  
      System::AnsiString::~AnsiString(&v22, 2);  
    }  
  }  
  while ( v15 == 4 );  
  return ExitCode == 0;  
} 
```

adddns可类似分析，这里不再赘述

## FASTDNS如何列出网卡的？
&emsp;&emsp;开始以为是找注册表，但是procmon无结果，后来还是根据字符串找到了下面的函数，该函数为设置dns的回调：
```C++
text:00403524 _TfrmFastDNS_actSetDNSExecute proc near ; DATA XREF: .data:fastdns___CPPdebugHook+1400 o
text:00403524 var_58          = dword ptr -58h  
text:00403524 var_54          = dword ptr -54h  
text:00403524 var_44          = word ptr -44h  
text:00403524 var_38          = dword ptr -38h  
text:00403524 var_30          = dword ptr -30h  
text:00403524 var_2C          = dword ptr -2Ch  
text:00403524 handle          = dword ptr -28h  
text:00403524 var_18          = dword ptr -18h  
text:00403524 var_10          = byte ptr -10h  
text:00403524 var_8           = byte ptr -8  
text:00403524 var_4           = byte ptr -4  
text:00403524  
text:00403524                 push    ebp 
text:00403525                 mov     ebp, esp 
text:00403527                 add     esp, 0FFFFFFA8h  
text:0040352A                 push    ebx 
text:0040352B                 push    esi 
text:0040352C                 push    edi 
text:0040352D                 mov     ebx, eax 
text:0040352F                 mov     eax, offset unk_4A29EC  
text:00403534                 call    @__InitExceptBlockLDTC  
text:00403539                 mov     [ebp+var_44], 8  
text:0040353F                 mov     edx, offset unk_4A22C4  
text:00403544                 lea     eax, [ebp+var_4]  
text:00403547                 call    @System@AnsiString@$bctr$qqrpxc ; System::AnsiString::AnsiString(char *)  
text:0040354C                 inc     [ebp+var_38]  
text:0040354F                 mov     edx, [eax]  
text:00403551                 mov     eax, [ebx+34Ch]  
text:00403557                 call    @TControl@SetText ; TControl::SetText  
text:0040355C                 dec     [ebp+var_38]  
text:0040355F                 lea     eax, [ebp+var_4]  
text:00403562                 mov     edx, 2  
text:00403567                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040356C                 xor     ecx, ecx 
text:0040356E                 mov     dl, 1  
text:00403570                 mov     [ebp+var_58], ecx 
text:00403573                 mov     ecx, ebx 
text:00403575                 mov     [ebp+var_44], 14h  
text:0040357B                 mov     eax, off_4A3D2C  
text:00403580                 call    sub_40738C  
text:00403585                 mov     [ebp+var_58], eax 
text:00403588                 mov     eax, [ebx+340h]  
text:0040358E                 call    @TCustomListView@GetSelection ; TCustomListView::GetSelection  
text:00403593                 test    eax, eax 
text:00403595                 jz      loc_403642  
text:0040359B                 mov     edx, [ebp+var_58]  
text:0040359E                 mov     eax, [edx+304h]  
text:004035A4                 mov     edx, [eax]  
text:004035A6                 call    dword ptr [edx+0D4h]  
text:004035AC                 mov     [ebp+var_44], 14h  
text:004035B2                 xor     esi, esi 
text:004035B4                 jmp     short loc_403629  
text:004035B6 ; ---------------------------------------------------------------------------  
text:004035B6  
text:004035B6 loc_4035B6:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+118 j  
text:004035B6                 mov     eax, [ebx+340h]  
text:004035BC                 mov     edx, esi 
text:004035BE                 mov     eax, [eax+22Ch]  
text:004035C4                 call    sub_48A1E4  
text:004035C9                 mov     edx, 3  
text:004035CE                 call    @TListItem@GetState ; TListItem::GetState  
text:004035D3                 test    al, al 
text:004035D5                 jz      short loc_403628  
text:004035D7                 mov     eax, [ebp+var_58]  
text:004035DA                 mov     edx, esi 
text:004035DC                 mov     edi, [eax+304h]  
text:004035E2                 mov     [ebp+var_44], 20h  
text:004035E8                 mov     eax, [ebx+340h]  
text:004035EE                 add     edi, 218h  
text:004035F4                 mov     eax, [eax+22Ch]  
text:004035FA                 call    sub_48A1E4  
text:004035FF                 mov     edx, eax 
text:00403601                 lea     eax, [ebp+var_8]  
text:00403604                 add     edx, 24h  
text:00403607                 call    @System@AnsiString@$bctr$qqrrx17System@AnsiString ; System::AnsiString::AnsiString(System::AnsiString &)  
text:0040360C                 inc     [ebp+var_38]  
text:0040360F                 mov     edx, [eax]  
text:00403611                 mov     eax, [edi]  
text:00403613                 mov     ecx, [eax]  
text:00403615                 call    dword ptr [ecx+38h]  
text:00403618                 dec     [ebp+var_38]  
text:0040361B                 lea     eax, [ebp+var_8]  
text:0040361E                 mov     edx, 2  
text:00403623                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:00403628  
text:00403628 loc_403628:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+B1 j  
text:00403628                 inc     esi 
text:00403629  
text:00403629 loc_403629:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+90 j  
text:00403629                 mov     ecx, [ebx+340h]  
text:0040362F                 mov     eax, [ecx+22Ch]  
text:00403635                 call    @TListItems@GetCount ; TListItems::GetCount  
text:0040363A                 cmp     esi, eax 
text:0040363C                 jl      loc_4035B6  
text:00403642  
text:00403642 loc_403642:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+71 j  
text:00403642                 mov     eax, [ebp+var_58]  
text:00403645                 mov     edx, [eax]  
text:00403647                 call    dword ptr [edx+0E8h]  
text:0040364D                 mov     ecx, [ebp+var_58]  
text:00403650                 cmp     byte ptr [ecx+308h], 0  
text:00403657                 jz      short loc_4036C3  
text:00403659                 mov     [ebp+var_44], 2Ch  
text:0040365F                 lea     eax, [ebp+var_10]  
text:00403662                 lea     edx, [ebp+handle]  
text:00403665                 push    eax 
text:00403666                 inc     [ebp+var_38]  
text:00403669                 push    offset aSDnsBg ; "设置DNS完成。"  
text:0040366E                 push    edx            ; _DWORD  
text:0040366F                 call    FormatString  
text:00403674                 add     esp, 0Ch  
text:00403677                 lea     ecx, [ebp+handle]  
text:0040367A                 add     [ebp+var_38], 4  
text:0040367E                 push    ecx 
text:0040367F                 call    MessageBox  
text:00403684                 pop     ecx 
text:00403685                 dec     [ebp+var_38]  
text:00403688                 mov     [ebp+var_44], 14h  
text:0040368E                 dec     [ebp+var_38]  
text:00403691                 mov     esi, [ebp+var_18]  
text:00403694                 mov     eax, [ebp+handle]  
text:00403697                 sub     esi, eax 
text:00403699                 mov     ebx, eax 
text:0040369B                 test    ebx, ebx 
text:0040369D                 jz      short loc_4036BA  
text:0040369F                 cmp     esi, 80h  
text:004036A5                 jbe     short loc_4036B0  
text:004036A7                 push    ebx            ; handle  
text:004036A8                 call    __rtl_close  
text:004036AD                 pop     ecx 
text:004036AE                 jmp     short loc_4036BA  
text:004036B0 ; ---------------------------------------------------------------------------  
text:004036B0  
text:004036B0 loc_4036B0:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+181 j  
text:004036B0                 push    esi 
text:004036B1                 push    ebx 
text:004036B2                 call    sub_4A0CA4  
text:004036B7                 add     esp, 8  
text:004036BA  
text:004036BA loc_4036BA:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+179 j  
text:004036BA                                        ; _TfrmFastDNS_actSetDNSExecute+18A j  
text:004036BA                 dec     [ebp+var_38]  
text:004036BD                 dec     [ebp+var_38]  
text:004036C0                 dec     [ebp+var_38]  
text:004036C3  
text:004036C3 loc_4036C3:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+133 j  
text:004036C3                 mov     [ebp+var_44], 0  
text:004036C9                 jmp     short loc_4036D6  
text:004036CB ; ---------------------------------------------------------------------------  
text:004036CB  
text:004036CB loc_4036CB:                            ; DATA XREF: .data:fastdns___CPPdebugHook+8F0 o  
text:004036CB                 mov     [ebp+var_44], 1Ch  
text:004036D1                 call    @_CatchCleanup$qv ; _CatchCleanup(void)  
text:004036D6  
text:004036D6 loc_4036D6:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+1A5 j  
text:004036D6                 mov     ebx, [ebp+var_58]  
text:004036D9                 mov     [ebp+var_30], ebx 
text:004036DC                 test    ebx, ebx 
text:004036DE                 jz      short loc_4036FE  
text:004036E0                 mov     eax, [ebx]  
text:004036E2                 mov     [ebp+var_2C], eax 
text:004036E5                 mov     [ebp+var_44], 5Ch  
text:004036EB                 mov     edx, 3  
text:004036F0                 mov     eax, [ebp+var_30]  
text:004036F3                 mov     ecx, [eax]  
text:004036F5                 call    dword ptr [ecx-4]  
text:004036F8                 mov     [ebp+var_44], 50h  
text:004036FE  
text:004036FE loc_4036FE:                            ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+1BA j  
text:004036FE                 mov     edx, [ebp+var_54]  
text:00403701                 mov     large fs:0, edx 
text:00403708                 pop     edi 
text:00403709                 pop     esi 
text:0040370A                 pop     ebx 
text:0040370B                 mov     esp, ebp 
text:0040370D                 pop     ebp 
text:0040370E                 retn 
text:0040370E _TfrmFastDNS_actSetDNSExecute endp 
t  
  
text:0040738C sub_40738C      proc near              ; CODE XREF: _TfrmFastDNS_actSetDNSExecute+5C p  
text:0040738C                                        ; DATA XREF: .data:fastdns___CPPdebugHook+1D0C o  
text:0040738C  
text:0040738C var_54          = byte ptr -54h  
text:0040738C var_49          = byte ptr -49h  
text:0040738C var_48          = dword ptr -48h  
text:0040738C var_38          = word ptr -38h  
text:0040738C var_2C          = dword ptr -2Ch  
text:0040738C var_28          = byte ptr -28h  
text:0040738C var_1C          = byte ptr -1Ch  
text:0040738C var_4           = dword ptr -4  
text:0040738C  
text:0040738C                 push    ebp 
text:0040738D                 mov     ebp, esp 
text:0040738F                 add     esp, 0FFFFFFACh  
text:00407392                 push    ebx 
text:00407393                 push    esi 
text:00407394                 push    edi 
text:00407395                 mov     [ebp+var_28], dl 
text:00407398                 test    dl, dl 
text:0040739A                 jle     short loc_4073A1  
text:0040739C                 call    __ClassCreate  
text:004073A1  
text:004073A1 loc_4073A1:                            ; CODE XREF: sub_40738C+E j  
text:004073A1                 mov     ebx, ecx 
text:004073A3                 mov     [ebp+var_49], dl 
text:004073A6                 mov     [ebp+var_4], eax 
text:004073A9                 lea     esi, [ebp+var_1C]  
text:004073AC                 mov     eax, offset unk_4A3A70  
text:004073B1                 call    @__InitExceptBlockLDTC  
text:004073B6                 mov     [ebp+var_38], 8  
text:004073BC                 mov     ecx, ebx 
text:004073BE                 xor     edx, edx 
text:004073C0                 mov     eax, [ebp+var_4]  
text:004073C3                 call    sub_401EFC  
text:004073C8                 add     [ebp+var_2C], 10h  
text:004073CC                 mov     edx, [ebp+var_4]  
text:004073CF                 xor     ecx, ecx 
text:004073D1                 xor     eax, eax 
text:004073D3                 mov     byte ptr [edx+308h], 0  
text:004073DA                 mov     [ebp+var_38], 20h  
text:004073E0                 inc     [ebp+var_2C]  
text:004073E3                 mov     [esi], ecx 
text:004073E5                 mov     [esi+4], eax 
text:004073E8                 xor     edx, edx 
text:004073EA                 inc     [ebp+var_2C]  
text:004073ED                 mov     [esi+10h], edx 
text:004073F0                 inc     [ebp+var_2C]  
text:004073F3                 inc     [ebp+var_2C]  
text:004073F6                 inc     [ebp+var_2C]  
text:004073F9                 dec     [ebp+var_2C]  
text:004073FC                 mov     [ebp+var_38], 14h  
text:00407402                 push    esi 
text:00407403                 call    sub_4071FC  
text:00407408                 pop     ecx 
text:00407409                 mov     ebx, [esi]  
text:0040740B                 jmp     short loc_407427  
text:0040740D ; ---------------------------------------------------------------------------  
text:0040740D  
text:0040740D loc_40740D:                            ; CODE XREF: sub_40738C+9E j  
text:0040740D                 mov     eax, [ebp+var_4]  
text:00407410                 mov     eax, [eax+300h]  
text:00407416                 add     eax, 218h  
text:0040741B                 mov     edx, [ebx]  
text:0040741D                 mov     eax, [eax]  
text:0040741F                 mov     ecx, [eax]  
text:00407421                 call    dword ptr [ecx+38h]  
text:00407424                 add     ebx, 4  
text:00407427  
text:00407427 loc_407427:                            ; CODE XREF: sub_40738C+7F j  
text:00407427                 cmp     ebx, [esi+4]  
text:0040742A                 jnz     short loc_40740D  
text:0040742C                 dec     [ebp+var_2C]  
text:0040742F                 lea     ecx, [ebp+var_54]  
text:00407432                 mov     eax, [esi+4]  
text:00407435                 mov     edx, [esi]  
text:00407437                 push    ecx 
text:00407438                 push    eax 
text:00407439                 push    edx 
text:0040743A                 call    sub_403254  
text:0040743F                 mov     [ebp+var_38], 14h  
text:00407445                 dec     [ebp+var_2C]  
text:00407448                 add     esp, 0Ch  
text:0040744B                 mov     edi, [esi+10h]  
text:0040744E                 mov     eax, [esi]  
text:00407450                 sub     edi, eax 
text:00407452                 test    edi, edi 
text:00407454                 jns     short loc_407459  
text:00407456                 add     edi, 3  
text:00407459  
text:00407459 loc_407459:                            ; CODE XREF: sub_40738C+C8 j  
text:00407459                 sar     edi, 2  
text:0040745C                 mov     ebx, eax 
text:0040745E                 test    ebx, ebx 
text:00407460                 jz      short loc_407482  
text:00407462                 mov     esi, edi 
text:00407464                 shl     esi, 2  
text:00407467                 cmp     esi, 80h  
text:0040746D                 jbe     short loc_407478  
text:0040746F                 push    ebx            ; handle  
text:00407470                 call    __rtl_close  
text:00407475                 pop     ecx 
text:00407476                 jmp     short loc_407482  
text:00407478 ; ---------------------------------------------------------------------------  
text:00407478  
text:00407478 loc_407478:                            ; CODE XREF: sub_40738C+E1 j  
text:00407478                 push    esi 
text:00407479                 push    ebx 
text:0040747A                 call    sub_4A0CA4  
text:0040747F                 add     esp, 8  
text:00407482  
text:00407482 loc_407482:                            ; CODE XREF: sub_40738C+D4 j  
text:00407482                                        ; sub_40738C+EA j  
text:00407482                 dec     [ebp+var_2C]  
text:00407485                 dec     [ebp+var_2C]  
text:00407488                 mov     [ebp+var_38], 8  
text:0040748E                 mov     eax, [ebp+var_48]  
text:00407491                 mov     large fs:0, eax 
text:00407497                 mov     eax, [ebp+var_4]  
text:0040749A                 cmp     [ebp+var_49], 0  
text:0040749E                 jz      short loc_4074A5  
text:004074A0                 call    @@AfterConstruction ; __AfterConstruction  
text:004074A5  
text:004074A5 loc_4074A5:                            ; CODE XREF: sub_40738C+112 j  
text:004074A5                 pop     edi 
text:004074A6                 pop     esi 
text:004074A7                 pop     ebx 
text:004074A8                 mov     esp, ebp 
text:004074AA                 pop     ebp 
text:004074AB                 retn 
text:004074AB sub_40738C      endp 
text:004074AB  
  
  
text:004071FC sub_4071FC      proc near              ; CODE XREF: showstatus+374 p  
text:004071FC                                        ; sub_40738C+77 p  
text:004071FC  
text:004071FC var_54          = byte ptr -54h  
text:004071FC var_4C          = dword ptr -4Ch  
text:004071FC SizePointer     = dword ptr -48h  
text:004071FC var_44          = byte ptr -44h  
text:004071FC var_3C          = byte ptr -3Ch  
text:004071FC var_34          = dword ptr -34h  
text:004071FC var_30          = dword ptr -30h  
text:004071FC var_20          = word ptr -20h  
text:004071FC var_14          = dword ptr -14h  
text:004071FC var_C           = dword ptr -0Ch  
text:004071FC var_8           = byte ptr -8  
text:004071FC var_4           = dword ptr -4  
text:004071FC arg_0           = dword ptr  8  
text:004071FC  
text:004071FC                 push    ebp 
text:004071FD                 mov     ebp, esp 
text:004071FF                 add     esp, 0FFFFFFACh  
text:00407202                 mov     eax, offset unk_4A39F0  
text:00407207                 push    ebx 
text:00407208                 push    esi 
text:00407209                 push    edi 
text:0040720A                 mov     ebx, [ebp+arg_0]  
text:0040720D                 call    @__InitExceptBlockLDTC  
text:00407212                 mov     edi, [ebx+4]  
text:00407215                 mov     esi, [ebx]  
text:00407217                 mov     eax, [ebx+4]  
text:0040721A                 lea     edx, [ebp+var_3C]  
text:0040721D                 mov     [ebp+var_34], eax 
text:00407220                 push    0  
text:00407222                 push    edx 
text:00407223                 push    esi 
text:00407224                 mov     ecx, [ebp+var_34]  
text:00407227                 push    ecx 
text:00407228                 push    edi 
text:00407229                 call    sub_403BF8  
text:0040722E                 add     esp, 14h  
text:00407231                 mov     edi, eax 
text:00407233                 mov     eax, [ebx+4]  
text:00407236                 mov     edx, edi 
text:00407238                 lea     ecx, [ebp+var_44]  
text:0040723B                 push    ecx 
text:0040723C                 push    eax 
text:0040723D                 push    edx 
text:0040723E                 call    sub_403254  
text:00407243                 add     esp, 0Ch  
text:00407246                 mov     [ebx+4], edi 
text:00407249                 mov     [ebp+var_20], 8  
text:0040724F                 push    280h  
text:00407254                 call    @$bnwa$qui     ; operator new[](uint)  
text:00407259                 pop     ecx 
text:0040725A                 mov     edi, eax 
text:0040725C                 mov     [ebp+SizePointer], 280h  
text:00407263                 mov     [ebp+var_20], 8  
text:00407269                 lea     eax, [ebp+SizePointer]  
text:0040726C                 push    eax            ; SizePointer  
text:0040726D                 push    edi            ; AdapterInfo  
text:0040726E                 call    GetAdaptersInfo  
text:00407273                 cmp     eax, 6Fh  
text:00407276                 jnz     short loc_40728B  
text:00407278                 push    edi            ; handle  
text:00407279                 call    __close  
text:0040727E                 pop     ecx 
text:0040727F                 mov     edx, [ebp+SizePointer]  
text:00407282                 push    edx 
text:00407283                 call    @$bnwa$qui     ; operator new[](uint)  
text:00407288                 pop     ecx 
text:00407289                 mov     edi, eax 
text:0040728B  
text:0040728B loc_40728B:                            ; CODE XREF: sub_4071FC+7A j  
text:0040728B                 lea     eax, [ebp+SizePointer]  
text:0040728E                 push    eax            ; SizePointer  
text:0040728F                 push    edi            ; AdapterInfo  
text:00407290                 call    GetAdaptersInfo  
text:00407295                 test    eax, eax 
text:00407297                 jnz     loc_40735D  
text:0040729D                 mov     esi, edi 
text:0040729F                 test    esi, esi 
text:004072A1                 jz      loc_40735D  
text:004072A7  
text:004072A7 loc_4072A7:                            ; CODE XREF: sub_4071FC+15B j  
text:004072A7                 mov     [ebp+var_20], 20h  
text:004072AD                 lea     edx, [esi+8]  
text:004072B0                 lea     eax, [ebp+var_8]  
text:004072B3                 call    @System@AnsiString@$bctr$qqrpxc ; System::AnsiString::AnsiString(char *)  
text:004072B8                 inc     [ebp+var_14]  
text:004072BB                 lea     ecx, [ebp+var_8]  
text:004072BE                 push    ecx 
text:004072BF                 xor     eax, eax 
text:004072C1                 mov     [ebp+var_4], eax 
text:004072C4                 lea     edx, [ebp+var_4]  
text:004072C7                 push    edx 
text:004072C8                 inc     [ebp+var_14]  
text:004072CB                 call    sub_406FD8  
text:004072D0                 add     esp, 8  
text:004072D3                 dec     [ebp+var_14]  
text:004072D6                 lea     eax, [ebp+var_8]  
text:004072D9                 mov     edx, 2  
text:004072DE                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:004072E3                 mov     [ebp+var_20], 14h  
text:004072E9                 cmp     [ebp+var_4], 0  
text:004072ED                 jz      short loc_40733D  
text:004072EF                 mov     ecx, [ebx+4]  
text:004072F2                 cmp     ecx, [ebx+10h]  
text:004072F5                 jz      short loc_407324  
text:004072F7                 mov     eax, [ebx+4]  
text:004072FA                 mov     [ebp+var_4C], eax 
text:004072FD                 mov     edx, [ebp+var_4C]  
text:00407300                 mov     [ebp+var_C], edx 
text:00407303                 test    edx, edx 
text:00407305                 jz      short loc_40731E  
text:00407307                 mov     [ebp+var_20], 38h  
text:0040730D                 lea     edx, [ebp+var_4]  
text:00407310                 mov     eax, [ebp+var_C]  
text:00407313                 call    @System@AnsiString@$bctr$qqrrx17System@AnsiString ; System::AnsiString::AnsiString(System::AnsiString &)  
text:00407318                 mov     [ebp+var_20], 2Ch  
text:0040731E  
text:0040731E loc_40731E:                            ; CODE XREF: sub_4071FC+109 j  
text:0040731E                 add     dword ptr [ebx+4], 4  
text:00407322                 jmp     short loc_40733D  
text:00407324 ; ---------------------------------------------------------------------------  
text:00407324  
text:00407324 loc_407324:                            ; CODE XREF: sub_4071FC+F9 j  
text:00407324                 push    1  
text:00407326                 push    1  
text:00407328                 lea     ecx, [ebp+var_54]  
text:0040732B                 push    ecx 
text:0040732C                 lea     eax, [ebp+var_4]  
text:0040732F                 push    eax 
text:00407330                 mov     edx, [ebx+4]  
text:00407333                 push    edx 
text:00407334                 push    ebx 
text:00407335                 call    sub_403C34  
text:0040733A                 add     esp, 18h  
text:0040733D  
text:0040733D loc_40733D:                            ; CODE XREF: sub_4071FC+F1 j  
text:0040733D                                        ; sub_4071FC+126 j  
text:0040733D                 mov     esi, [esi]  
text:0040733F                 dec     [ebp+var_14]  
text:00407342                 lea     eax, [ebp+var_4]  
text:00407345                 mov     edx, 2  
text:0040734A                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040734F                 mov     [ebp+var_20], 8  
text:00407355                 test    esi, esi 
text:00407357                 jnz     loc_4072A7  
text:0040735D  
text:0040735D loc_40735D:                            ; CODE XREF: sub_4071FC+9B j  
text:0040735D                                        ; sub_4071FC+A5 j  
text:0040735D                 mov     [ebp+var_20], 0  
text:00407363                 jmp     short loc_407372  
text:00407365 ; ---------------------------------------------------------------------------  
text:00407365  
text:00407365 loc_407365:                            ; DATA XREF: .data:fastdns___CPPdebugHook+1904 o  
text:00407365                 xor     edi, edi 
text:00407367                 mov     [ebp+var_20], 10h  
text:0040736D                 call    @_CatchCleanup$qv ; _CatchCleanup(void)  
text:00407372  
text:00407372 loc_407372:                            ; CODE XREF: sub_4071FC+167 j  
text:00407372                 push    edi            ; handle  
text:00407373                 call    __close  
text:00407378                 pop     ecx 
text:00407379                 mov     eax, [ebp+var_30]  
text:0040737C                 mov     large fs:0, eax 
text:00407382                 pop     edi 
text:00407383                 pop     esi 
text:00407384                 pop     ebx 
text:00407385                 mov     esp, ebp 
text:00407387                 pop     ebp 
text:00407388                 retn 
```

&emsp;&emsp;这里可以看到是用GetAdapterInfo得到的，得到的每个AdapterName再送到sub_406FDB中检查：
```C++
int __cdecl sub_406FD8(int a1, int a2)  
{  
  __InitExceptBlockLDTC();  
  v15 = 0;  
  v7 = TRegistry::`...'(dword_427AA0, 1, 131097);  
  TRegistry::SetRootKey(v7, -2147483646);  
  v2 = System::AnsiString::AnsiString(  
         &v14,  
         "SYSTEM\\CurrentControlSet\\Control\\Network\\{4D36E972-E325-11CE-BFC1-08002BE10318}\\");  
  v13 = 0;  
  System::AnsiString::operator+(v2, a2, &v13);  
  System::AnsiString::AnsiString(&v12, "\\Connection\\");  
  v11 = 0;  
  System::AnsiString::operator+(&v13, &v12, &v11);  
  v3 = TRegistry::OpenKey(v7, v11, 0);  
  System::AnsiString::~AnsiString(&v11, 2);  
  System::AnsiString::~AnsiString(&v12, 2);  
  System::AnsiString::~AnsiString(&v13, 2);  
  System::AnsiString::~AnsiString(&v14, 2);  
  if ( v3 )  
  {  
    v4 = *(_DWORD *)System::AnsiString::AnsiString(&v10, "Name");  
    v9 = 0;  
    TRegistry::ReadString(v7, v4);  
    sub_4A0484(&v15, &v9);  
    System::AnsiString::~AnsiString(&v9, 2);  
    System::AnsiString::~AnsiString(&v10, 2);  
  }  
  else 
  {  
    System::AnsiString::AnsiString(&v8, &unk_4A37E9);  
    sub_4A0484(&v15, &v8);  
    System::AnsiString::~AnsiString(&v8, 2);  
  }  
  if ( v7 )  
  {  
    v5 = *(_DWORD *)v7;  
    (*(void (__fastcall **)(int, signed int))(*(_DWORD *)v7 - 4))(v7, 3);  
  }  
  sub_4A0484(a1, &v15);  
  System::AnsiString::~AnsiString(&v15, 2);  
  return a1;  
} 
```

&emsp;&emsp;做一些字符串操作然后查注册表中存储的网卡信息，取得name并存储

## FASTDNS如何初始化那些DNS串的，存储在哪里？
&emsp;&emsp;熟悉windows消息机制的会清楚，ListView显示条目用的是SendMessage LVM_SETITEXT或类似机制，而绝不是自己设置，那么我们只需要在启动时加载程序并拦截该函数，看看数目和ip数目吻合程度就知道了。由于结果较多只贴出循环节：
```Txt
hwnd=812a0 msg=4103 
hwnd=812a0 msg=4101 
hwnd=812a0 msg=4109 
hwnd=812a0 msg=4142 
```
&emsp;&emsp;此时可以用spy++查下hwnd是不是ListView，然后看看消息代码，可知分别对应：`LVM_GETITEMA LVM_INSERTITEMA LVM_FINDITEMA LVM_SETITEMTEXTA`，第二个和第四个正是我们处理多column的ListView所用的操作，没想到Delphi程序也是这么做的（只不过可能经过层层封装）。下条件断点，进入后发现LVITEM.pszText为-1(LPSTR_TEXTCALLBACK，该消息是说显示到哪再加载那里的数据，较为灵活，对分析造成了困难)，也就是说并未再此时赋值，因此使用该方法寻找dns地址根源是失败的，那么我们再次想到字符串，用WinHex搜索地址串，并下数据断点，等移动ListView的滚动条时，程序自动断下，进入`.text:0048C33C ; TCustomListView::CNNotify`
```C++
else 
{  
  v4 = v3 + 308;  
  if ( v4 )  
  {  
    v5 = v4 - 151;  
    if ( v5 )  
    {  
      v6 = v5 - 5;  
      if ( v6 )  
      {  
        if ( v6 == 2 )                        // LVN_GETDISPINFOA  
        {  
          v64 = TCustomListView::GetItem(v106, (const void *)(v2 + 12));  
          v65 = *(_DWORD *)(v105 + 8);  
          v66 = v65 + 12;  
          if ( *(_BYTE *)(v65 + 12) & 1 )  
          {  
            v67 = *(_DWORD *)(v65 + 20);  
            if ( v67 )  
            {  
              v96 = *(_DWORD *)(v64 + 8);  
              if ( v67 > (*(int (__cdecl **)(unsigned int, _UNKNOWN *, int *))(*(_DWORD *)v96 + 20))(v87, v88, v89) )  
              {  
                **(_BYTE **)(v66 + 20) = 0;  
              }  
              else 
              {  
                (*(void (__fastcall **)(int, int, int *))(*(_DWORD *)v96 + 12))(v96, *(_DWORD *)(v66 + 8) - 1, &v91);  
                StrPLCopy(*(void **)(v66 + 20), v91, *(_DWORD *)(v66 + 24) - 1);  
              }  
            }  
            else 
            {  
              StrPLCopy(*(void **)(v65 + 32), *(char **)(v64 + 0x24), *(_DWORD *)(v65 + 36) - 1);  
            }  
          } 
```

&emsp;&emsp;可以看到这里正是我们需要的LVN_GETDISPINFOA，程序通过该回调设置真正的列表数据，程序经过最后面这个StrPLCopy，因此我们的重点是看谁改了v64，向上发现源头在`TCustomListView::GetItem`，跟进，发现最终源头为`SendMessageA(hWnd, LVM_GETITEM, 0, &lvitem);`得到的`*(char*)(lvitem->lparam+0x24)`。最终结果。那么谁设置了lparam呢？继续追寻
```Txt
-100 -101 -102 -150
LVN_ITEMCHANGING FFFFFF9C
LVN_ITEMCHANGED  FFFFFF9B
LVN_INSERTITEM   FFFFFF9A
LVN_GETDISPINFOA FFFFFF6A
```

&emsp;&emsp;如果下消息断点，并不能发现LVM_SETITEM消息，可能是对delphi不了解吧，这时还要采用最古老的方法，既然显示给我们了，那么内存里已经有字符串，并且一定是解密的，winhex找到位置后，下硬件断点，到了这里：
```
FastDNS!DnslistFinalize+0x555:
0040ba8d 3a4d14          cmp     cl,byte ptr [ebp+14h]      ss:002b:0018fbf4=2c
```

&emsp;&emsp;向上层回溯，看看拿到dns地址的根源在什么地方：
```C++
text:0040A258 sub_40A258      proc near              ; CODE XREF: sub_402474+81 p  
text:0040A258  
text:0040A258                 push    ebp 
text:0040A259                 mov     ebp, esp 
text:0040A25B                 add     esp, 0FFFFFE80h  
text:0040A261                 push    ebx 
text:0040A262                 push    esi 
text:0040A263                 push    edi 
text:0040A264                 lea     ebx, [ebp+var_114]  
text:0040A26A                 mov     eax, offset unk_4B2A5C  
text:0040A26F                 call    @__InitExceptBlockLDTC  
text:0040A274                 mov     edx, [ebp+arg_0]  
text:0040A277                 push    edx 
text:0040A278                 call    sub_40C368  
text:0040A27D                 pop     ecx 
text:0040A27E                 xor     eax, eax 
text:0040A280                 mov     word ptr [ebx+10h], 14h  
text:0040A286                 inc     dword ptr [ebx+1Ch]  
text:0040A289                 lea     ecx, [ebp-38h]  
text:0040A28C                 mov     [ebp+var_118], ecx 
text:0040A292                 mov     [ebp+var_18], eax 
text:0040A295                 xor     edx, edx 
text:0040A297                 xor     ecx, ecx 
text:0040A299                 mov     [ebp+var_18+4], edx 
text:0040A29C                 lea     eax, [ebp-40h]  
text:0040A29F                 inc     dword ptr [ebx+1Ch]  
text:0040A2A2                 mov     [ebp+var_18+10h], ecx 
text:0040A2A5                 inc     dword ptr [ebx+1Ch]  
text:0040A2A8                 inc     dword ptr [ebx+1Ch]  
text:0040A2AB                 inc     dword ptr [ebx+1Ch]  
text:0040A2AE                 dec     dword ptr [ebx+1Ch]  
text:0040A2B1                 lea     edx, [ebp+handle]  
text:0040A2B4                 mov     word ptr [ebx+10h], 8  
text:0040A2BA                 mov     word ptr [ebx+10h], 2Ch  
text:0040A2C0                 push    eax 
text:0040A2C1                 inc     dword ptr [ebx+1Ch]  
text:0040A2C4                 push    offset asc_4B2712 ; "\r\n"  
text:0040A2C9                 push    edx            ; _DWORD  
text:0040A2CA                 call    FormatString  
text:0040A2CF                 add     dword ptr [ebx+1Ch], 4  
text:0040A2D3                 mov     ecx, [ebp+var_18+4]  
text:0040A2D6                 add     esp, 0Ch  
text:0040A2D9                 cmp     ecx, [ebp+var_18+10h]  
text:0040A2DC                 jz      short loc_40A316  
text:0040A2DE                 mov     eax, [ebp+var_18+4]  
text:0040A2E1                 mov     [ebp+var_11C], eax 
text:0040A2E7                 mov     edx, [ebp+var_11C]  
text:0040A2ED                 mov     [ebp-5Ch], edx 
text:0040A2F0                 test    edx, edx 
text:0040A2F2                 jz      short loc_40A310  
text:0040A2F4                 mov     word ptr [ebx+10h], 38h  
text:0040A2FA                 lea     eax, [ebp+handle]  
text:0040A2FD                 push    eax 
text:0040A2FE                 mov     ecx, [ebp-5Ch]  
text:0040A301                 push    ecx 
text:0040A302                 call    sub_40AD98  
text:0040A307                 mov     word ptr [ebx+10h], 2Ch  
text:0040A30D                 add     esp, 8  
text:0040A310  
text:0040A310 loc_40A310:                            ; CODE XREF: sub_40A258+9A j  
text:0040A310                 add     [ebp+var_18+4], 18h  
text:0040A314                 jmp     short loc_40A335  
text:0040A316 ; ---------------------------------------------------------------------------  
text:0040A316  
text:0040A316 loc_40A316:                            ; CODE XREF: sub_40A258+84 j  
text:0040A316                 push    1  
text:0040A318                 push    1  
text:0040A31A                 lea     eax, [ebp+var_124]  
text:0040A320                 push    eax 
text:0040A321                 lea     edx, [ebp+handle]  
text:0040A324                 push    edx 
text:0040A325                 mov     ecx, [ebp+var_18+4]  
text:0040A328                 push    ecx 
text:0040A329                 lea     eax, [ebp+var_18]  
text:0040A32C                 push    eax 
text:0040A32D                 call    sub_40A968  
text:0040A332                 add     esp, 18h  
text:0040A335  
text:0040A335 loc_40A335:                            ; CODE XREF: sub_40A258+BC j  
text:0040A335                 dec     dword ptr [ebx+1Ch]  
text:0040A338                 mov     word ptr [ebx+10h], 8  
text:0040A33E                 dec     dword ptr [ebx+1Ch]  
text:0040A341                 mov     edi, [ebp+handle+10h]  
text:0040A344                 sub     edi, [ebp+handle]  
text:0040A347                 mov     esi, [ebp+handle]  
text:0040A34A                 test    esi, esi 
text:0040A34C                 jz      short loc_40A369  
text:0040A34E                 cmp     edi, 80h  
text:0040A354                 jbe     short loc_40A35F  
text:0040A356                 push    esi            ; handle  
text:0040A357                 call    __rtl_close  
text:0040A35C                 pop     ecx 
text:0040A35D                 jmp     short loc_40A369  
text:0040A35F ; ---------------------------------------------------------------------------  
text:0040A35F  
text:0040A35F loc_40A35F:                            ; CODE XREF: sub_40A258+FC j  
text:0040A35F                 push    edi 
text:0040A360                 push    esi 
text:0040A361                 call    sub_4A0CA4  
text:0040A366                 add     esp, 8  
text:0040A369  
text:0040A369 loc_40A369:                            ; CODE XREF: sub_40A258+F4 j  
text:0040A369                                        ; sub_40A258+105 j  
text:0040A369                 dec     dword ptr [ebx+1Ch]  
text:0040A36C                 dec     dword ptr [ebx+1Ch]  
text:0040A36F                 dec     dword ptr [ebx+1Ch]  
text:0040A372                 lea     eax, [ebp-64h]  
text:0040A375                 mov     word ptr [ebx+10h], 5Ch  
text:0040A37B                 push    eax 
text:0040A37C                 lea     edx, [ebp+var_7C]  
text:0040A37F                 inc     dword ptr [ebx+1Ch]  
text:0040A382                 push    offset asc_4B2715 ; "\r"  
text:0040A387                 push    edx            ; _DWORD  
text:0040A388                 call    FormatString  
text:0040A38D                 add     dword ptr [ebx+1Ch], 4  
text:0040A391                 mov     ecx, [ebp+var_18+4]  
text:0040A394                 add     esp, 0Ch  
text:0040A397                 cmp     ecx, [ebp+var_18+10h]  
text:0040A39A                 jz      short loc_40A3D4  
text:0040A39C                 mov     eax, [ebp+var_18+4]  
text:0040A39F                 mov     [ebp+var_128], eax 
text:0040A3A5                 mov     edx, [ebp+var_128]  
text:0040A3AB                 mov     [ebp-80h], edx 
text:0040A3AE                 test    edx, edx 
text:0040A3B0                 jz      short loc_40A3CE  
text:0040A3B2                 mov     word ptr [ebx+10h], 68h  
text:0040A3B8                 lea     eax, [ebp+var_7C]  
text:0040A3BB                 push    eax 
text:0040A3BC                 mov     ecx, [ebp-80h]  
text:0040A3BF                 push    ecx 
text:0040A3C0                 call    sub_40AD98  
text:0040A3C5                 mov     word ptr [ebx+10h], 5Ch  
text:0040A3CB                 add     esp, 8  
text:0040A3CE  
text:0040A3CE loc_40A3CE:                            ; CODE XREF: sub_40A258+158 j  
text:0040A3CE                 add     [ebp+var_18+4], 18h  
text:0040A3D2                 jmp     short loc_40A3F3  
text:0040A3D4 ; ---------------------------------------------------------------------------  
text:0040A3D4  
text:0040A3D4 loc_40A3D4:                            ; CODE XREF: sub_40A258+142 j  
text:0040A3D4                 push    1  
text:0040A3D6                 push    1  
text:0040A3D8                 lea     eax, [ebp+var_130]  
text:0040A3DE                 push    eax 
text:0040A3DF                 lea     edx, [ebp+var_7C]  
text:0040A3E2                 push    edx 
text:0040A3E3                 mov     ecx, [ebp+var_18+4]  
text:0040A3E6                 push    ecx 
text:0040A3E7                 lea     eax, [ebp+var_18]  
text:0040A3EA                 push    eax 
text:0040A3EB                 call    sub_40A968  
text:0040A3F0                 add     esp, 18h  
text:0040A3F3  
text:0040A3F3 loc_40A3F3:                            ; CODE XREF: sub_40A258+17A j  
text:0040A3F3                 dec     dword ptr [ebx+1Ch]  
text:0040A3F6                 mov     word ptr [ebx+10h], 8  
text:0040A3FC                 dec     dword ptr [ebx+1Ch]  
text:0040A3FF                 mov     edi, [ebp+var_7C+10h]  
text:0040A402                 sub     edi, [ebp+var_7C]  
text:0040A405                 mov     esi, [ebp+var_7C]  
text:0040A408                 test    esi, esi 
text:0040A40A                 jz      short loc_40A427  
text:0040A40C                 cmp     edi, 80h  
text:0040A412                 jbe     short loc_40A41D  
text:0040A414                 push    esi            ; handle  
text:0040A415                 call    __rtl_close  
text:0040A41A                 pop     ecx 
text:0040A41B                 jmp     short loc_40A427  
text:0040A41D ; ---------------------------------------------------------------------------  
text:0040A41D  
text:0040A41D loc_40A41D:                            ; CODE XREF: sub_40A258+1BA j  
text:0040A41D                 push    edi 
text:0040A41E                 push    esi 
text:0040A41F                 call    sub_4A0CA4  
text:0040A424                 add     esp, 8  
text:0040A427  
text:0040A427 loc_40A427:                            ; CODE XREF: sub_40A258+1B2 j  
text:0040A427                                        ; sub_40A258+1C3 j  
text:0040A427                 dec     dword ptr [ebx+1Ch]  
text:0040A42A                 dec     dword ptr [ebx+1Ch]  
text:0040A42D                 dec     dword ptr [ebx+1Ch]  
text:0040A430                 lea     eax, [ebp-88h]  
text:0040A436                 mov     word ptr [ebx+10h], 8Ch  
text:0040A43C                 push    eax 
text:0040A43D                 lea     edx, [ebp+var_A0]  
text:0040A443                 inc     dword ptr [ebx+1Ch]  
text:0040A446                 push    offset asc_4B2717 ; "\n"  
text:0040A44B                 push    edx            ; _DWORD  
text:0040A44C                 call    FormatString  
text:0040A451                 add     dword ptr [ebx+1Ch], 4  
text:0040A455                 mov     ecx, [ebp+var_18+4]  
text:0040A458                 add     esp, 0Ch  
text:0040A45B                 cmp     ecx, [ebp+var_18+10h]  
text:0040A45E                 jz      short loc_40A4A1  
text:0040A460                 mov     eax, [ebp+var_18+4]  
text:0040A463                 mov     [ebp+var_134], eax 
text:0040A469                 mov     edx, [ebp+var_134]  
text:0040A46F                 mov     [ebp+var_A4], edx 
text:0040A475                 test    edx, edx 
text:0040A477                 jz      short loc_40A49B  
text:0040A479                 mov     word ptr [ebx+10h], 98h  
text:0040A47F                 lea     eax, [ebp+var_A0]  
text:0040A485                 push    eax 
text:0040A486                 mov     ecx, [ebp+var_A4]  
text:0040A48C                 push    ecx 
text:0040A48D                 call    sub_40AD98  
text:0040A492                 mov     word ptr [ebx+10h], 8Ch  
text:0040A498                 add     esp, 8  
text:0040A49B  
text:0040A49B loc_40A49B:                            ; CODE XREF: sub_40A258+21F j  
text:0040A49B                 add     [ebp+var_18+4], 18h  
text:0040A49F                 jmp     short loc_40A4C3  
text:0040A4A1 ; ---------------------------------------------------------------------------  
text:0040A4A1  
text:0040A4A1 loc_40A4A1:                            ; CODE XREF: sub_40A258+206 j  
text:0040A4A1                 push    1  
text:0040A4A3                 push    1  
text:0040A4A5                 lea     eax, [ebp+var_13C]  
text:0040A4AB                 push    eax 
text:0040A4AC                 lea     edx, [ebp+var_A0]  
text:0040A4B2                 push    edx 
text:0040A4B3                 mov     ecx, [ebp+var_18+4]  
text:0040A4B6                 push    ecx 
text:0040A4B7                 lea     eax, [ebp+var_18]  
text:0040A4BA                 push    eax 
text:0040A4BB                 call    sub_40A968  
text:0040A4C0                 add     esp, 18h  
text:0040A4C3  
text:0040A4C3 loc_40A4C3:                            ; CODE XREF: sub_40A258+247 j  
text:0040A4C3                 dec     dword ptr [ebx+1Ch]  
text:0040A4C6                 mov     word ptr [ebx+10h], 8  
text:0040A4CC                 dec     dword ptr [ebx+1Ch]  
text:0040A4CF                 mov     edi, [ebp+var_A0+10h]  
text:0040A4D5                 sub     edi, [ebp+var_A0]  
text:0040A4DB                 mov     esi, [ebp+var_A0]  
text:0040A4E1                 test    esi, esi 
text:0040A4E3                 jz      short loc_40A500  
text:0040A4E5                 cmp     edi, 80h  
text:0040A4EB                 jbe     short loc_40A4F6  
text:0040A4ED                 push    esi            ; handle  
text:0040A4EE                 call    __rtl_close  
text:0040A4F3                 pop     ecx 
text:0040A4F4                 jmp     short loc_40A500  
text:0040A4F6 ; ---------------------------------------------------------------------------  
text:0040A4F6  
text:0040A4F6 loc_40A4F6:                            ; CODE XREF: sub_40A258+293 j  
text:0040A4F6                 push    edi 
text:0040A4F7                 push    esi 
text:0040A4F8                 call    sub_4A0CA4  
text:0040A4FD                 add     esp, 8  
text:0040A500  
text:0040A500 loc_40A500:                            ; CODE XREF: sub_40A258+28B j  
text:0040A500                                        ; sub_40A258+29C j  
text:0040A500                 dec     dword ptr [ebx+1Ch]  
text:0040A503                 dec     dword ptr [ebx+1Ch]  
text:0040A506                 dec     dword ptr [ebx+1Ch]  
text:0040A509                 lea     eax, [ebp+var_AC]  
text:0040A50F                 mov     word ptr [ebx+10h], 0BCh  
text:0040A515                 inc     dword ptr [ebx+1Ch]  
text:0040A518                 mov     [ebp+var_140], eax 
text:0040A51E                 xor     edx, edx 
text:0040A520                 xor     ecx, ecx 
text:0040A522                 mov     [ebp+var_30], edx 
text:0040A525                 mov     [ebp+var_30+4], ecx 
text:0040A528                 inc     dword ptr [ebx+1Ch]  
text:0040A52B                 xor     eax, eax 
text:0040A52D                 mov     [ebp+var_30+10h], eax 
text:0040A530                 lea     edx, [ebp+var_30]  
text:0040A533                 inc     dword ptr [ebx+1Ch]  
text:0040A536                 inc     dword ptr [ebx+1Ch]  
text:0040A539                 inc     dword ptr [ebx+1Ch]  
text:0040A53C                 dec     dword ptr [ebx+1Ch]  
text:0040A53F                 lea     ecx, [ebp+var_18]  
text:0040A542                 mov     word ptr [ebx+10h], 8  
text:0040A548                 push    edx 
text:0040A549                 push    ecx 
text:0040A54A                 mov     eax, [ebp+arg_4]  
text:0040A54D                 push    eax 
text:0040A54E                 call    sub_40B7F4  
text:0040A553                 add     esp, 0Ch  
text:0040A556                 mov     edi, [ebp+var_30]  
text:0040A559                 jmp     loc_40A850  
text:0040A55E ; ---------------------------------------------------------------------------  
text:0040A55E  
text:0040A55E loc_40A55E:                            ; CODE XREF: sub_40A258+5FB j  
text:0040A55E                 push    28h  
text:0040A560                 call    @$bnew$qui     ; operator new(uint)  
text:0040A565                 pop     ecx 
text:0040A566                 mov     [ebp+var_B0], eax 
text:0040A56C                 test    eax, eax 
text:0040A56E                 jz      short loc_40A5E6  
text:0040A570                 mov     word ptr [ebx+10h], 0E0h  
text:0040A576                 mov     edx, [ebp+var_B0]  
text:0040A57C                 xor     ecx, ecx 
text:0040A57E                 mov     [edx+14h], ecx 
text:0040A581                 mov     eax, offset off_4B2DD4  
text:0040A586                 mov     edx, [ebp+var_B0]  
text:0040A58C                 mov     [edx+18h], eax 
text:0040A58F                 mov     ecx, [ebp+var_B0]  
text:0040A595                 xor     eax, eax 
text:0040A597                 mov     [ecx+1Ch], eax 
text:0040A59A                 mov     edx, offset off_4B2DE4  
text:0040A59F                 mov     ecx, [ebp+var_B0]  
text:0040A5A5                 mov     [ecx+18h], edx 
text:0040A5A8                 mov     esi, [ebp+var_B0]  
text:0040A5AE                 add     esi, 20h  
text:0040A5B1                 xor     eax, eax 
text:0040A5B3                 mov     [esi], eax 
text:0040A5B5                 inc     dword ptr [ebx+1Ch]  
text:0040A5B8                 mov     edx, [ebp+var_B0]  
text:0040A5BE                 add     edx, 24h  
text:0040A5C1                 mov     [ebp+var_144], edx 
text:0040A5C7                 mov     ecx, [ebp+var_144]  
text:0040A5CD                 xor     eax, eax 
text:0040A5CF                 mov     [ecx], eax 
text:0040A5D1                 inc     dword ptr [ebx+1Ch]  
text:0040A5D4                 add     dword ptr [ebx+1Ch], 0FFFFFFFEh  
text:0040A5D8                 mov     word ptr [ebx+10h], 0D4h  
text:0040A5DE                 mov     edx, [ebp+var_B0]  
text:0040A5E4                 jmp     short loc_40A5EC  
text:0040A5E6 ; ---------------------------------------------------------------------------  
text:0040A5E6  
text:0040A5E6 loc_40A5E6:                            ; CODE XREF: sub_40A258+316 j  
text:0040A5E6                 mov     edx, [ebp+var_B0]  
text:0040A5EC  
text:0040A5EC loc_40A5EC:                            ; CODE XREF: sub_40A258+38C j  
text:0040A5EC                 mov     esi, edx 
text:0040A5EE                 mov     word ptr [ebx+10h], 8  
text:0040A5F4                 mov     word ptr [ebx+10h], 0ECh  
text:0040A5FA                 push    2Ch  
text:0040A5FC                 push    0  
text:0040A5FE                 push    edi 
text:0040A5FF                 lea     eax, [ebp+var_C8]  
text:0040A605                 push    eax 
text:0040A606                 call    sub_40B96C  
text:0040A60B                 add     esp, 10h  
text:0040A60E                 lea     ecx, [ebp+var_C8]  
text:0040A614                 add     dword ptr [ebx+1Ch], 4  
text:0040A618                 mov     [ebp+var_148], ecx 
text:0040A61E                 mov     eax, [ebp+var_148]  
text:0040A624                 mov     edx, [eax]  
text:0040A626                 lea     eax, [ebp+var_CC]  
text:0040A62C                 call    @System@AnsiString@$bctr$qqrpxc ; System::AnsiString::AnsiString(char *)  
text:0040A631                 inc     dword ptr [ebx+1Ch]  
text:0040A634                 lea     edx, [ebp+var_CC]  
text:0040A63A                 lea     eax, [esi+20h]  
text:0040A63D                 call    sub_4A0484  
text:0040A642                 dec     dword ptr [ebx+1Ch]  
text:0040A645                 lea     eax, [ebp+var_CC]  
text:0040A64B                 mov     edx, 2  
text:0040A650                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040A655                 dec     dword ptr [ebx+1Ch]  
text:0040A658                 mov     word ptr [ebx+10h], 8  
text:0040A65E                 dec     dword ptr [ebx+1Ch]  
text:0040A661                 mov     ecx, [ebp+var_B8]  
text:0040A667                 mov     eax, [ebp+var_C8]  
text:0040A66D                 sub     ecx, eax 
text:0040A66F                 mov     [ebp+var_14C], ecx 
text:0040A675                 mov     [ebp+var_150], eax 
text:0040A67B                 cmp     [ebp+var_150], 0  
text:0040A682                 jz      short loc_40A6B5  
text:0040A684                 cmp     [ebp+var_14C], 80h  
text:0040A68E                 jbe     short loc_40A69F  
text:0040A690                 mov     edx, [ebp+var_150]  
text:0040A696                 push    edx            ; handle  
text:0040A697                 call    __rtl_close  
text:0040A69C                 pop     ecx 
text:0040A69D                 jmp     short loc_40A6B5  
text:0040A69F ; ---------------------------------------------------------------------------  
text:0040A69F  
text:0040A69F loc_40A69F:                            ; CODE XREF: sub_40A258+436 j  
text:0040A69F                 mov     ecx, [ebp+var_14C]  
text:0040A6A5                 push    ecx 
text:0040A6A6                 mov     eax, [ebp+var_150]  
text:0040A6AC                 push    eax 
text:0040A6AD                 call    sub_4A0CA4  
text:0040A6B2                 add     esp, 8  
text:0040A6B5  
text:0040A6B5 loc_40A6B5:                            ; CODE XREF: sub_40A258+42A j  
text:0040A6B5                                        ; sub_40A258+445 j  
text:0040A6B5                 dec     dword ptr [ebx+1Ch]  
text:0040A6B8                 dec     dword ptr [ebx+1Ch]  
text:0040A6BB                 xor     edx, edx 
text:0040A6BD                 mov     word ptr [ebx+10h], 110h  
text:0040A6C3                 mov     [ebp+var_D0], edx 
text:0040A6C9                 lea     edx, [ebp+var_D0]  
text:0040A6CF                 inc     dword ptr [ebx+1Ch]  
text:0040A6D2                 lea     eax, [esi+20h]  
text:0040A6D5                 call    sub_4A05EC  
text:0040A6DA                 lea     edx, [ebp+var_D0]  
text:0040A6E0                 lea     eax, [esi+20h]  
text:0040A6E3                 call    sub_4A0484  
text:0040A6E8                 dec     dword ptr [ebx+1Ch]  
text:0040A6EB                 lea     eax, [ebp+var_D0]  
text:0040A6F1                 mov     edx, 2  
text:0040A6F6                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040A6FB                 mov     word ptr [ebx+10h], 11Ch  
text:0040A701                 push    2Ch  
text:0040A703                 push    1  
text:0040A705                 push    edi 
text:0040A706                 lea     ecx, [ebp+var_E8]  
text:0040A70C                 push    ecx 
text:0040A70D                 call    sub_40B96C  
text:0040A712                 add     esp, 10h  
text:0040A715                 lea     eax, [ebp+var_E8]  
text:0040A71B                 add     dword ptr [ebx+1Ch], 4  
text:0040A71F                 mov     [ebp+var_154], eax 
text:0040A725                 lea     eax, [ebp+var_EC]  
text:0040A72B                 mov     edx, [ebp+var_154]  
text:0040A731                 mov     edx, [edx]  
text:0040A733                 call    @System@AnsiString@$bctr$qqrpxc ; System::AnsiString::AnsiString(char *)  
text:0040A738                 inc     dword ptr [ebx+1Ch]  
text:0040A73B                 lea     edx, [ebp+var_EC]  
text:0040A741                 lea     eax, [esi+24h]  
text:0040A744                 call    sub_4A0484  
text:0040A749                 dec     dword ptr [ebx+1Ch]  
text:0040A74C                 lea     eax, [ebp+var_EC]  
text:0040A752                 mov     edx, 2  
text:0040A757                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040A75C                 dec     dword ptr [ebx+1Ch]  
text:0040A75F                 mov     word ptr [ebx+10h], 8  
text:0040A765                 dec     dword ptr [ebx+1Ch]  
text:0040A768                 mov     ecx, [ebp+var_D8]  
text:0040A76E                 mov     eax, [ebp+var_E8]  
text:0040A774                 sub     ecx, eax 
text:0040A776                 mov     [ebp+var_158], ecx 
text:0040A77C                 mov     [ebp+var_15C], eax 
text:0040A782                 cmp     [ebp+var_15C], 0  
text:0040A789                 jz      short loc_40A7BC  
text:0040A78B                 cmp     [ebp+var_158], 80h  
text:0040A795                 jbe     short loc_40A7A6  
text:0040A797                 mov     edx, [ebp+var_15C]  
text:0040A79D                 push    edx            ; handle  
text:0040A79E                 call    __rtl_close  
text:0040A7A3                 pop     ecx 
text:0040A7A4                 jmp     short loc_40A7BC  
text:0040A7A6 ; ---------------------------------------------------------------------------  
text:0040A7A6  
text:0040A7A6 loc_40A7A6:                            ; CODE XREF: sub_40A258+53D j  
text:0040A7A6                 mov     ecx, [ebp+var_158]  
text:0040A7AC                 push    ecx 
text:0040A7AD                 mov     eax, [ebp+var_15C]  
text:0040A7B3                 push    eax 
text:0040A7B4                 call    sub_4A0CA4  
text:0040A7B9                 add     esp, 8  
text:0040A7BC  
text:0040A7BC loc_40A7BC:                            ; CODE XREF: sub_40A258+531 j  
text:0040A7BC                                        ; sub_40A258+54C j  
text:0040A7BC                 dec     dword ptr [ebx+1Ch]  
text:0040A7BF                 dec     dword ptr [ebx+1Ch]  
text:0040A7C2                 xor     edx, edx 
text:0040A7C4                 mov     [esi+8], edx 
text:0040A7C7                 mov     [esi+0Ch], edx 
text:0040A7CA                 lea     eax, [esi+20h]  
text:0040A7CD                 mov     byte ptr [esi+10h], 1  
text:0040A7D1                 cmp     dword ptr [eax], 0  
text:0040A7D4                 jz      short loc_40A7DA  
text:0040A7D6                 mov     edx, [eax]  
text:0040A7D8                 jmp     short loc_40A7DF  
text:0040A7DA ; ---------------------------------------------------------------------------  
text:0040A7DA  
text:0040A7DA loc_40A7DA:                            ; CODE XREF: sub_40A258+57C j  
text:0040A7DA                 mov     edx, offset unk_4B2719  
text:0040A7DF  
text:0040A7DF loc_40A7DF:                            ; CODE XREF: sub_40A258+580 j  
text:0040A7DF                 push    edx            ; cp  
text:0040A7E0                 call    inet_addr  
text:0040A7E5                 cmp     eax, 0FFFFFFFFh  
text:0040A7E8                 jz      short loc_40A7F9  
text:0040A7EA                 push    esi 
text:0040A7EB                 mov     eax, [ebp+arg_0]  
text:0040A7EE                 push    eax 
text:0040A7EF                 call    sub_40C46C  
text:0040A7F4                 add     esp, 8  
text:0040A7F7                 jmp     short loc_40A84D  
text:0040A7F9 ; ---------------------------------------------------------------------------  
text:0040A7F9  
text:0040A7F9 loc_40A7F9:                            ; CODE XREF: sub_40A258+590 j  
text:0040A7F9                 mov     [ebp+var_F0], esi 
text:0040A7FF                 cmp     [ebp+var_F0], 0  
text:0040A806                 jz      short loc_40A84D  
text:0040A808                 mov     word ptr [ebx+10h], 14Ch  
text:0040A80E                 dec     dword ptr [ebx+1Ch]  
text:0040A811                 mov     edx, 2  
text:0040A816                 mov     eax, [ebp+var_F0]  
text:0040A81C                 add     eax, 24h  
text:0040A81F                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040A824                 dec     dword ptr [ebx+1Ch]  
text:0040A827                 mov     edx, 2  
text:0040A82C                 mov     eax, [ebp+var_F0]  
text:0040A832                 add     eax, 20h  
text:0040A835                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040A83A                 mov     ecx, [ebp+var_F0]  
text:0040A840                 push    ecx            ; handle  
text:0040A841                 call    __rtl_close  
text:0040A846                 pop     ecx 
text:0040A847                 mov     word ptr [ebx+10h], 140h  
text:0040A84D  
text:0040A84D loc_40A84D:                            ; CODE XREF: sub_40A258+59F j  
text:0040A84D                                        ; sub_40A258+5AE j  
text:0040A84D                 add     edi, 18h  
text:0040A850  
text:0040A850 loc_40A850:                            ; CODE XREF: sub_40A258+301 j  
text:0040A850                 cmp     edi, [ebp+var_30+4]  
text:0040A853                 jnz     loc_40A55E  
text:0040A859                 dec     dword ptr [ebx+1Ch]  
text:0040A85C                 lea     ecx, [ebp+var_164]  
text:0040A862                 mov     eax, [ebp+var_30+4]  
text:0040A865                 mov     edx, [ebp+var_30]  
text:0040A868                 push    ecx 
text:0040A869                 push    eax 
text:0040A86A                 push    edx 
text:0040A86B                 call    sub_40AB5C  
text:0040A870                 mov     word ptr [ebx+10h], 8  
text:0040A876                 dec     dword ptr [ebx+1Ch]  
text:0040A879                 mov     ecx, 18h  
text:0040A87E                 mov     eax, [ebp+var_30+10h]  
text:0040A881                 add     esp, 0Ch  
text:0040A884                 sub     eax, [ebp+var_30]  
text:0040A887                 cdq 
text:0040A888                 idiv    ecx 
text:0040A88A                 mov     [ebp+var_168], eax 
text:0040A890                 mov     esi, [ebp+var_30]  
text:0040A893                 test    esi, esi 
text:0040A895                 jz      short loc_40A8CE  
text:0040A897                 mov     eax, [ebp+var_168]  
text:0040A89D                 shl     eax, 3  
text:0040A8A0                 lea     eax, [eax+eax*2]  
text:0040A8A3                 mov     [ebp+var_16C], eax 
text:0040A8A9                 cmp     [ebp+var_16C], 80h  
text:0040A8B3                 jbe     short loc_40A8BE  
text:0040A8B5                 push    esi            ; handle  
text:0040A8B6                 call    __rtl_close  
text:0040A8BB                 pop     ecx 
text:0040A8BC                 jmp     short loc_40A8CE  
text:0040A8BE ; ---------------------------------------------------------------------------  
text:0040A8BE  
text:0040A8BE loc_40A8BE:                            ; CODE XREF: sub_40A258+65B j  
text:0040A8BE                 mov     edx, [ebp+var_16C]  
text:0040A8C4                 push    edx 
text:0040A8C5                 push    esi 
text:0040A8C6                 call    sub_4A0CA4  
text:0040A8CB                 add     esp, 8  
text:0040A8CE  
text:0040A8CE loc_40A8CE:                            ; CODE XREF: sub_40A258+63D j  
text:0040A8CE                                        ; sub_40A258+664 j  
text:0040A8CE                 dec     dword ptr [ebx+1Ch]  
text:0040A8D1                 dec     dword ptr [ebx+1Ch]  
text:0040A8D4                 dec     dword ptr [ebx+1Ch]  
text:0040A8D7                 mov     eax, [ebp+var_18+4]  
text:0040A8DA                 mov     edx, [ebp+var_18]  
text:0040A8DD                 mov     [ebp+var_170], edx 
text:0040A8E3                 lea     ecx, [ebp+var_178]  
text:0040A8E9                 push    ecx 
text:0040A8EA                 push    eax 
text:0040A8EB                 mov     eax, [ebp+var_170]  
text:0040A8F1                 push    eax 
text:0040A8F2                 call    sub_40AB5C  
text:0040A8F7                 add     esp, 0Ch  
text:0040A8FA                 dec     dword ptr [ebx+1Ch]  
text:0040A8FD                 mov     eax, [ebp+var_18+10h]  
text:0040A900                 sub     eax, [ebp+var_18]  
text:0040A903                 mov     ecx, 18h  
text:0040A908                 cdq 
text:0040A909                 idiv    ecx 
text:0040A90B                 mov     [ebp+var_17C], eax 
text:0040A911                 mov     esi, [ebp+var_18]  
text:0040A914                 test    esi, esi 
text:0040A916                 jz      short loc_40A94F  
text:0040A918                 mov     eax, [ebp+var_17C]  
text:0040A91E                 shl     eax, 3  
text:0040A921                 lea     eax, [eax+eax*2]  
text:0040A924                 mov     [ebp+var_180], eax 
text:0040A92A                 cmp     [ebp+var_180], 80h  
text:0040A934                 jbe     short loc_40A93F  
text:0040A936                 push    esi            ; handle  
text:0040A937                 call    __rtl_close  
text:0040A93C                 pop     ecx 
text:0040A93D                 jmp     short loc_40A94F  
text:0040A93F ; ---------------------------------------------------------------------------  
text:0040A93F  
text:0040A93F loc_40A93F:                            ; CODE XREF: sub_40A258+6DC j  
text:0040A93F                 mov     edx, [ebp+var_180]  
text:0040A945                 push    edx 
text:0040A946                 push    esi 
text:0040A947                 call    sub_4A0CA4  
text:0040A94C                 add     esp, 8  
text:0040A94F  
text:0040A94F loc_40A94F:                            ; CODE XREF: sub_40A258+6BE j  
text:0040A94F                                        ; sub_40A258+6E5 j  
text:0040A94F                 dec     dword ptr [ebx+1Ch]  
text:0040A952                 dec     dword ptr [ebx+1Ch]  
text:0040A955                 mov     ecx, [ebx]  
text:0040A957                 mov     large fs:0, ecx 
text:0040A95E                 pop     edi 
text:0040A95F                 pop     esi 
text:0040A960                 pop     ebx 
text:0040A961                 mov     esp, ebp 
text:0040A963                 pop     ebp 
text:0040A964                 retn 
```

&emsp;&emsp;这里已经在做字符串处理了，内存中是4.2.2.1,美国 科罗拉多州布隆菲尔德市Level 3通信公司这种形式，因此需要用逗号吧地址和位置分开，查看变量的访问情况后（这是个很耗时的过程）发现还要向上回溯：
```C++
text:00402474 sub_402474      proc near              ; CODE XREF: sub_401B74+1A8 p  
text:00402474  
text:00402474 var_48          = byte ptr -48h  
text:00402474 s               = dword ptr -24h  
text:00402474 var_20          = byte ptr -20h  
text:00402474 handle          = dword ptr -18h  
text:00402474 var_8           = dword ptr -8  
text:00402474 arg_0           = dword ptr  8  
text:00402474  
text:00402474                 push    ebp 
text:00402475                 mov     ebp, esp 
text:00402477                 add     esp, 0FFFFFFB8h  
text:0040247A                 mov     eax, offset unk_4A2564  
text:0040247F                 push    ebx 
text:00402480                 push    esi 
text:00402481                 push    edi 
text:00402482                 lea     edi, [ebp+var_48]  
text:00402485                 mov     ebx, [ebp+arg_0]  
text:00402488                 call    @__InitExceptBlockLDTC  
text:0040248D                 mov     word ptr [edi+10h], 14h  
text:00402493                 lea     edx, [ebp+var_20]  
text:00402496                 xor     ecx, ecx 
text:00402498                 push    edx 
text:00402499                 lea     eax, [ebp+s]  
text:0040249C                 inc     dword ptr [edi+1Ch]  
text:0040249F                 mov     [ebp+s], ecx 
text:004024A2                 push    eax 
text:004024A3                 inc     dword ptr [edi+1Ch]  
text:004024A6                 call    sub_408F5C  
text:004024AB                 cmp     [ebp+s], 0  
text:004024AF                 pop     ecx 
text:004024B0                 jz      short loc_4024B7  
text:004024B2                 mov     edx, [ebp+s]  
text:004024B5                 jmp     short loc_4024BC  
text:004024B7 ; ---------------------------------------------------------------------------  
text:004024B7  
text:004024B7 loc_4024B7:                            ; CODE XREF: sub_402474+3C j  
text:004024B7                 mov     edx, offset unk_4A21CB  
text:004024BC  
text:004024BC loc_4024BC:                            ; CODE XREF: sub_402474+41 j  
text:004024BC                 push    edx            ; s  
text:004024BD                 lea     eax, [ebp+handle]  
text:004024C0                 push    eax            ; _DWORD  
text:004024C1                 call    FormatString  
text:004024C6                 add     dword ptr [edi+1Ch], 4  
text:004024CA                 dec     dword ptr [edi+1Ch]  
text:004024CD                 add     esp, 0Ch  
text:004024D0                 dec     dword ptr [edi+1Ch]  
text:004024D3                 lea     eax, [ebp+s]  
text:004024D6                 mov     edx, 2  
text:004024DB                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:004024E0                 mov     word ptr [edi+10h], 8  
text:004024E6                 mov     eax, [ebx+400h]  
text:004024EC                 test    eax, eax 
text:004024EE                 jz      short loc_4024FD  
text:004024F0                 lea     edx, [ebp+handle]  
text:004024F3                 push    edx 
text:004024F4                 push    eax 
text:004024F5                 call    sub_40A258  
text:004024FA                 add     esp, 8  
text:004024FD  
text:004024FD loc_4024FD:                            ; CODE XREF: sub_402474+7A j  
text:004024FD                 dec     dword ptr [edi+1Ch]  
text:00402500                 dec     dword ptr [edi+1Ch]  
text:00402503                 mov     esi, [ebp+var_8]  
text:00402506                 mov     eax, [ebp+handle]  
text:00402509                 sub     esi, eax 
text:0040250B                 mov     ebx, eax 
text:0040250D                 test    ebx, ebx 
text:0040250F                 jz      short loc_40252C  
text:00402511                 cmp     esi, 80h  
text:00402517                 jbe     short loc_402522  
text:00402519                 push    ebx            ; handle  
text:0040251A                 call    __rtl_close  
text:0040251F                 pop     ecx 
text:00402520                 jmp     short loc_40252C  
text:00402522 ; ---------------------------------------------------------------------------  
text:00402522  
text:00402522 loc_402522:                            ; CODE XREF: sub_402474+A3 j  
text:00402522                 push    esi 
text:00402523                 push    ebx 
text:00402524                 call    sub_4A0CA4  
text:00402529                 add     esp, 8  
text:0040252C  
text:0040252C loc_40252C:                            ; CODE XREF: sub_402474+9B j  
text:0040252C                                        ; sub_402474+AC j  
text:0040252C                 dec     dword ptr [edi+1Ch]  
text:0040252F                 dec     dword ptr [edi+1Ch]  
text:00402532                 mov     eax, [edi]  
text:00402534                 mov     large fs:0, eax 
text:0040253A                 pop     edi 
text:0040253B                 pop     esi 
text:0040253C                 pop     ebx 
text:0040253D                 mov     esp, ebp 
text:0040253F                 pop     ebp 
text:00402540                 retn 
text:0040254  
```
  
&emsp;&emsp;再次调试，发现408F5C处得到的s已经包含地址，而之前再无修改s之处或者传递之处，因此确定sub_408F5C用于解密字符串，之所以是解密，是因为无法通过资源和字符串找到这些数据，也就说一定经过加密了。进入该函数，这下有趣了：  
```C++
text:00408F5C sub_408F5C      proc near              ; CODE XREF: sub_402474+32 p  
text:00408F5C  
text:00408F5C var_28          = dword ptr -28h  
text:00408F5C var_18          = word ptr -18h  
text:00408F5C var_C           = dword ptr -0Ch  
text:00408F5C var_4           = dword ptr -4  
text:00408F5C arg_0           = dword ptr  8  
text:00408F5C  
text:00408F5C                 push    ebp 
text:00408F5D                 mov     ebp, esp 
text:00408F5F                 add     esp, 0FFFFFFD8h  
text:00408F62                 mov     eax, offset unk_4B1FA0  
text:00408F67                 call    @__InitExceptBlockLDTC  
text:00408F6C                 mov     [ebp+var_18], 8  
text:00408F72                 push    0D3EEh  
text:00408F77                 push    offset unk_4A4B80  
text:00408F7C                 xor     edx, edx 
text:00408F7E                 mov     [ebp+var_4], edx 
text:00408F81                 lea     ecx, [ebp+var_4]  
text:00408F84                 push    ecx 
text:00408F85                 inc     [ebp+var_C]  
text:00408F88                 call    decode  
text:00408F8D                 add     esp, 0Ch  
text:00408F90                 lea     edx, [ebp+var_4]  
text:00408F93                 mov     eax, [ebp+arg_0]  
text:00408F96                 call    sub_4A0484  
text:00408F9B                 mov     eax, [ebp+arg_0]  
text:00408F9E                 mov     edx, 2  
text:00408FA3                 mov     [ebp+var_18], 14h  
text:00408FA9                 push    eax 
text:00408FAA                 lea     eax, [ebp+var_4]  
text:00408FAD                 dec     [ebp+var_C]  
text:00408FB0                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:00408FB5                 pop     eax 
text:00408FB6                 mov     [ebp+var_18], 8  
text:00408FBC                 inc     [ebp+var_C]  
text:00408FBF                 mov     edx, [ebp+var_28]  
text:00408FC2                 mov     large fs:0, edx 
text:00408FC9                 mov     esp, ebp 
text:00408FCB                 pop     ebp 
text:00408FCC                 retn 
text:00408FCC sub_408F5C      endp  
   
text:0040CA68 ; int __cdecl decode(int dest, int src, int len)  
text:0040CA68 decode          proc near              ; CODE XREF: dosetdns+7B p  
text:0040CA68                                        ; sub_4082F4+75 p ...  
text:0040CA68  
text:0040CA68 var_30          = dword ptr -30h  
text:0040CA68 var_2C          = dword ptr -2Ch  
text:0040CA68 var_1C          = word ptr -1Ch  
text:0040CA68 var_10          = dword ptr -10h  
text:0040CA68 var_8           = dword ptr -8  
text:0040CA68 var_4           = byte ptr -4  
text:0040CA68 dest            = dword ptr  8  
text:0040CA68 src             = dword ptr  0Ch  
text:0040CA68 len             = dword ptr  10h  
text:0040CA68  
text:0040CA68                 push    ebp 
text:0040CA69                 mov     ebp, esp 
text:0040CA6B                 add     esp, 0FFFFFFD0h  
text:0040CA6E                 mov     eax, offset unk_4B3420  
text:0040CA73                 call    @__InitExceptBlockLDTC  
text:0040CA78                 mov     [ebp+var_1C], 8  
text:0040CA7E                 lea     eax, [ebp+var_4]  
text:0040CA81                 call    unknown_libname_1128 ; CBuilder 5 runtime  
text:0040CA86                 inc     [ebp+var_10]  
text:0040CA89                 mov     [ebp+var_1C], 14h  
text:0040CA8F                 xor     edx, edx 
text:0040CA91                 mov     [ebp+var_30], edx 
text:0040CA94                 mov     ecx, [ebp+var_30]  
text:0040CA97                 cmp     ecx, [ebp+len]  
text:0040CA9A                 jge     short loc_40CAE4  
text:0040CA9C  
text:0040CA9C loc_40CA9C:                            ; CODE XREF: decode+7A j  
text:0040CA9C                 mov     [ebp+var_1C], 20h  
text:0040CAA2                 mov     eax, [ebp+var_30]  
text:0040CAA5                 mov     edx, [ebp+src]  
text:0040CAA8                 xor     ecx, ecx 
text:0040CAAA                 mov     cl, [edx+eax]  
text:0040CAAD                 mov     dl, byte_4B32CC[ecx]  
text:0040CAB3                 lea     eax, [ebp+var_8]  
text:0040CAB6                 call    @System@AnsiString@$bctr$qqrc ; System::AnsiString::AnsiString(char)  
text:0040CABB                 inc     [ebp+var_10]  
text:0040CABE                 lea     edx, [ebp+var_8]  
text:0040CAC1                 lea     eax, [ebp+var_4]  
text:0040CAC4                 call    sub_4A0498  
text:0040CAC9                 dec     [ebp+var_10]  
text:0040CACC                 lea     eax, [ebp+var_8]  
text:0040CACF                 mov     edx, 2  
text:0040CAD4                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040CAD9                 inc     [ebp+var_30]  
text:0040CADC                 mov     ecx, [ebp+var_30]  
text:0040CADF                 cmp     ecx, [ebp+len]  
text:0040CAE2                 jl      short loc_40CA9C  
text:0040CAE4  
text:0040CAE4 loc_40CAE4:                            ; CODE XREF: decode+32 j  
text:0040CAE4                 mov     [ebp+var_1C], 2Ch  
text:0040CAEA                 lea     edx, [ebp+var_4]  
text:0040CAED                 mov     eax, [ebp+dest]  
text:0040CAF0                 call    sub_4A0484  
text:0040CAF5                 mov     eax, [ebp+dest]  
text:0040CAF8                 mov     [ebp+var_1C], 38h  
text:0040CAFE                 push    eax 
text:0040CAFF                 dec     [ebp+var_10]  
text:0040CB02                 lea     eax, [ebp+var_4]  
text:0040CB05                 mov     edx, 2  
text:0040CB0A                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:0040CB0F                 pop     eax 
text:0040CB10                 mov     [ebp+var_1C], 2Ch  
text:0040CB16                 inc     [ebp+var_10]  
text:0040CB19                 mov     edx, [ebp+var_2C]  
text:0040CB1C                 mov     large fs:0, edx 
text:0040CB23                 mov     esp, ebp 
text:0040CB25                 pop     ebp 
text:0040CB26                 retn 
```

&emsp;&emsp;4A4B80和4B32CC处在data段，可以看到作者煞费苦心地构造了2个数组，后者用于做ascii码可逆映射，前者是加密过的字符串数组。大小分别为54254和256，既然如此，我们写程序总不能把这些数据从文件提取出来再解密啊，所以还是用winhex拷贝来的快。这里只是一种好奇心，看看作者到底怎么存储和解密这些数据。

## FASTDNS怎么测网速的呢？也就是最核心的部分？
&emsp;&emsp;根据前述，核心在TfrmFastDNS_actTestSpendingExecute中。
```Txt
+00h virtualtable
    +0 STOP
    +4 FUNC1
    +8 FUNC2
+04h HANDLE hthread
+08h ????
+18h FARPROC func0
+1Ch LPVOID param0
+20h FARPROC func1//OnIdle处理函数
+24h LPVOID param1
+28h BOOL IsMainthread??
+30h struct*
     +00h int leftTime//剩余次数
     +08h double pertime//平均消耗时间
     +10h flag statu  //成功0 失败3
+38h FARPROC callbackfunc  //测试结束的回调函数showstatus
+3Ch DWORD param2                //callbackfunc的参数
+40h int repeatTime//取样次数
+44h int timeout//超时
+48h Class IdDNSResolver_TIdDNSResolver* extends TIdUDPBase extends TIdUDPBase extends....
//原来delphi自带了组件，好方便啊，不过我得自行解决
        +00h virtualtable 00412B24
                TIdUDPBase::AssignError
                TIdUDPBase:efineProperties
                TIdUDPBase::Assign
                TIdUDPBase:oaded
                。。。。。。。。。。
        +74h char* ipdns
        +7Ch int timeout
        +90h package*//发包结构体
        +94h int repeattime
        +9Ch int sendinfo//要发送的数据包
        。。。。。。。。        
+50h char* ipstart//存放ip地址字串起始位置
+54h char* ipend//存放ip地址字串结束位置
```

```Asm
text:0040970C ; DWORD __cdecl FUNC2(int structa, int *usedtime)  
text:0040970C FUNC2           proc near               ; DATA XREF: .data:fastdns___CPPdebugHook+105D4 o  
text:0040970C  
text:0040970C var_2C          = dword ptr -2Ch  
text:0040970C var_1C          = word ptr -1Ch  
text:0040970C var_10          = dword ptr -10h  
text:0040970C var_8           = byte ptr -8  
text:0040970C var_4           = byte ptr -4  
text:0040970C structa         = dword ptr  8  
text:0040970C usedtime        = dword ptr  0Ch  
text:0040970C  
text:0040970C                 push    ebp  
text:0040970D                 mov     ebp, esp  
text:0040970F                 add     esp, 0FFFFFFD4h  
text:00409712                 push    ebx  
text:00409713                 push    esi  
text:00409714                 push    edi  
text:00409715                 mov     edi, [ebp+usedtime]  
text:00409718                 mov     ebx, [ebp+structa]  
text:0040971B                 mov     eax, offset unk_4B25EC  
text:00409720                 call    @__InitExceptBlockLDTC  
text:00409725                 mov     [ebp+var_1C], 8  
text:0040972B                 mov     esi, [ebx+48h]  
text:0040972E                 test    esi, esi  
text:00409730                 jz      loc_40984A  
text:00409736                 mov     [ebp+var_1C], 14h  
text:0040973C                 mov     edx, [ebx+50h]  
text:0040973F                 lea     eax, [ebp+var_4]  
text:00409742                 call    @System@AnsiString@$bctr$qqrpxc ; System::AnsiString::AnsiString(char *)  
text:00409747                 mov     edx, eax  
text:00409749                 inc     [ebp+var_10]  
text:0040974C                 mov     eax, esi  
text:0040974E                 add     eax, 74h  
text:00409751                 call    sub_4A0484  
text:00409756                 dec     [ebp+var_10]  
text:00409759                 lea     eax, [ebp+var_4]  
text:0040975C                 mov     edx, 2  
text:00409761                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:00409766                 mov     ecx, [ebx+44h]  
text:00409769                 mov     eax, [ebx+48h]  
text:0040976C                 mov     [eax+7Ch], ecx  
text:0040976F                 mov     edx, [eax]  
text:00409771                 call    dword ptr [edx+48h]  
text:00409774                 mov     ecx, [ebx+48h]  
text:00409777                 xor     edx, edx  
text:00409779                 mov     eax, [ecx+90h]  
text:0040977F                 call    sub_412E58  
text:00409784                 mov     ecx, [ebx+48h]  
text:00409787                 xor     edx, edx  
text:00409789                 mov     eax, [ecx+90h]  
text:0040978F                 call    sub_412E6C  
text:00409794                 mov     ecx, [ebx+48h]  
text:00409797                 mov     dl, 1  
text:00409799                 mov     eax, [ecx+90h]  
text:0040979F                 call    sub_412E88  
text:004097A4                 mov     eax, [ebx+48h]  
text:004097A7                 mov     edx, [eax+90h]  
text:004097AD                 mov     word ptr [edx+0Ch], 1  
text:004097B3                 mov     eax, [eax+94h]  
text:004097B9                 call    @TCollection@Clear ; TCollection::Clear  
text:004097BE                 mov     edx, [ebx+48h]  
text:004097C1                 mov     eax, [edx+94h]  
text:004097C7                 call    sub_412EA8  
text:004097CC                 mov     [ebp+var_1C], 8  
text:004097D2                 mov     [ebp+var_1C], 20h  
text:004097D8                 mov     esi, eax  
text:004097DA                 lea     eax, [ebp+var_8]  
text:004097DD                 mov     edx, offset aWww_microsoft_ ; "www.microsoft.com" 
text:004097E2                 call    @System@AnsiString@$bctr$qqrpxc ; System::AnsiString::AnsiString(char *)  
text:004097E7                 inc     [ebp+var_10]  
text:004097EA                 lea     edx, [ebp+var_8]  
text:004097ED                 lea     eax, [esi+10h]  
text:004097F0                 call    sub_4A0484  
text:004097F5                 dec     [ebp+var_10]  
text:004097F8                 lea     eax, [ebp+var_8]  
text:004097FB                 mov     edx, 2  
text:00409800                 call    @System@AnsiString@$bdtr$qqrv ; System::AnsiString::~AnsiString(void)  
text:00409805                 mov     word ptr [esi+14h], 1  
text:0040980B                 mov     word ptr [esi+0Ch], 1  
text:00409811                 call    @GetCurrentTime  
text:00409816                 mov     esi, eax  
text:00409818                 mov     [ebp+var_1C], 8  
text:0040981E                 mov     eax, [ebx+48h]  
text:00409821                 call    sub_413174  
text:00409826                 call    @GetCurrentTime  
text:0040982B                 mov     ebx, eax  
text:0040982D                 sub     ebx, esi  
text:0040982F                 mov     [edi], ebx  
text:00409831                 cmp     ebx, 1  
text:00409834                 jnb     short loc_40983C  
text:00409836                 mov     dword ptr [edi], 1  
text:0040983C  
text:0040983C loc_40983C:                             ; CODE XREF: FUNC2+128 j  
text:0040983C                 mov     al, 1  
text:0040983E                 mov     edx, [ebp+var_2C]  
text:00409841                 mov     large fs:0, edx  
text:00409848                 jmp     short loc_409869  
text:0040984A ; ---------------------------------------------------------------------------  
text:0040984A  
text:0040984A loc_40984A:                             ; CODE XREF: FUNC2+24 j  
text:0040984A                 mov     [ebp+var_1C], 0  
text:00409850                 jmp     short loc_40985D  
text:00409852 ; ---------------------------------------------------------------------------  
text:00409852  
text:00409852 loc_409852:                             ; DATA XREF: .data:fastdns___CPPdebugHook+1051C o  
text:00409852                 mov     [ebp+var_1C], 10h  
text:00409858                 call    @_CatchCleanup$qv ; _CatchCleanup(void)  
text:0040985D  
text:0040985D loc_40985D:                             ; CODE XREF: FUNC2+144 j  
text:0040985D                 xor     eax, eax  
text:0040985F                 mov     edx, [ebp+var_2C]  
text:00409862                 mov     large fs:0, edx  
text:00409869  
text:00409869 loc_409869:                             ; CODE XREF: FUNC2+13C j  
text:00409869                 pop     edi  
text:0040986A                 pop     esi  
text:0040986B                 pop     ebx  
text:0040986C                 mov     esp, ebp  
text:0040986E                 pop     ebp  
text:0040986F                 retn 
```

本段程序构造某网络包类，sub_413174为真正执行代码，在其前后取时间进行测速：

```Asm
text:00413174 sub_413174      proc near               ; CODE XREF: FUNC2+115 p  
text:00413174  
text:00413174 var_8           = dword ptr -8  
text:00413174 var_4           = dword ptr -4  
text:00413174  
text:00413174                 push    ebp  
text:00413175                 mov     ebp, esp  
text:00413177                 add     esp, 0FFFFFFF8h  
text:0041317A                 xor     edx, edx  
text:0041317C                 mov     [ebp+var_8], edx  
text:0041317F                 mov     [ebp+var_4], eax  
text:00413182                 xor     eax, eax  
text:00413184                 push    ebp  
text:00413185                 push    offset loc_413208  
text:0041318A                 push    dword ptr fs:[eax]  
text:0041318D                 mov     fs:[eax], esp  
text:00413190                 xor     eax, eax  
text:00413192                 push    ebp  
text:00413193                 push    offset loc_4131EB  
text:00413198                 push    dword ptr fs:[eax]  
text:0041319B                 mov     fs:[eax], esp  
text:0041319E                 mov     eax, [ebp+var_4]  
text:004131A1                 call    sub_413470  
text:004131A6                 mov     eax, [ebp+var_4]  
text:004131A9                 mov     edx, [eax+9Ch]  
text:004131AF                 mov     eax, [ebp+var_4]  
text:004131B2                 call    @TIdUDPClient@Send ; TIdUDPClient::Send  
text:004131B7                 lea     ecx, [ebp+var_8]  
text:004131BA                 or      edx, 0FFFFFFFFh  
text:004131BD                 mov     eax, [ebp+var_4]  
text:004131C0                 call    @TIdUDPBase@ReceiveString ; TIdUDPBase::ReceiveString  
text:004131C5                 mov     edx, [ebp+var_8]  
text:004131C8                 mov     eax, [ebp+var_4]  
text:004131CB                 add     eax, 0A0h  
text:004131D0                 call    @@LStrAsg       ; __linkproc__ LStrAsg  
text:004131D5                 xor     eax, eax  
text:004131D7                 pop     edx  
text:004131D8                 pop     ecx  
text:004131D9                 pop     ecx  
text:004131DA                 mov     fs:[eax], edx  
text:004131DD                 push    offset loc_4131F2  
text:004131E2  
text:004131E2 loc_4131E2:                             ; CODE XREF: sub_413174+7C j  
text:004131E2                 mov     eax, [ebp+var_4]  
text:004131E5                 call    sub_414B3C  
text:004131EA                 retn  
text:004131EB ; ---------------------------------------------------------------------------  
text:004131EB  
text:004131EB loc_4131EB:                             ; DATA XREF: sub_413174+1F o  
text:004131EB                 jmp     @@HandleFinally ; __linkproc__ HandleFinally  
text:004131F0 ; ---------------------------------------------------------------------------  
text:004131F0                 jmp     short loc_4131E2  
text:004131F2 ; ---------------------------------------------------------------------------  
text:004131F2  
text:004131F2 loc_4131F2:                             ; CODE XREF: sub_413174+76 j  
text:004131F2                                         ; DATA XREF: sub_413174+69 o  
text:004131F2                 xor     eax, eax  
text:004131F4                 pop     edx  
text:004131F5                 pop     ecx  
text:004131F6                 pop     ecx  
text:004131F7                 mov     fs:[eax], edx  
text:004131FA                 push    offset loc_41320F  
text:004131FF  
text:004131FF loc_4131FF:                             ; CODE XREF: sub_413174+99 j  
text:004131FF                 lea     eax, [ebp+var_8]  
text:00413202                 call    @@LStrClr       ; __linkproc__ LStrClr  
text:00413207                 retn  
text:00413208 ; ---------------------------------------------------------------------------  
text:00413208  
text:00413208 loc_413208:                             ; DATA XREF: sub_413174+11 o  
text:00413208                 jmp     @@HandleFinally ; __linkproc__ HandleFinally  
text:0041320D ; ---------------------------------------------------------------------------  
text:0041320D                 jmp     short loc_4131FF  
text:0041320F ; ---------------------------------------------------------------------------  
text:0041320F  
text:0041320F loc_41320F:                             ; CODE XREF: sub_413174+93 j  
text:0041320F                                         ; DATA XREF: sub_413174+86 o  
text:0041320F                 pop     ecx  
text:00413210                 pop     ecx  
text:00413211                 pop     ebp  
text:00413212                 retn 
```

&emsp;&emsp;从以上代码可以得知，重要调用有：
* sub_413470
* TIdDNSResolver::Send
* TIdDNSResolver::ReceiveString
* sub_414B3C  

&emsp;&emsp;由于调用函数库较多，较为浪费时间，因此这里用WireShark抓取+API break的方式查看网络情况，可以得到如下结果：

启动时：对每个ip做inet_addr   
检测时：WSAStartup 101   
&emsp;&emsp;socket af=AF_INET type=SOCK_DGRAM protocol=IPPROTO_IP   
&emsp;&emsp;setsockopt level=SOL_SOCKET optname=SO_BROADCAST   optval="\0\0\0\0" optlen=4  
&emsp;&emsp;ntohs hostshort=53//DNS  
&emsp;&emsp;sendto len=35 flags=0 tolen=sizeof(sockaddr) buf=   
&emsp;&emsp;&emsp;&emsp;b4 c3 01 00 00 01 00 00-00 00 00 00 03 77 77 77  .............www  
&emsp;&emsp;&emsp;&emsp;09 6d 69 63 72 6f 73 6f-66 74 03 63 6f 6d 00 00  .microsoft.com..  
&emsp;&emsp;&emsp;&emsp;01 00 01                            
&emsp;&emsp;&emsp;&emsp;sockaddr=02 00 00 35 04 02 02 01-00 00 00 00 00 00 00 00  
&emsp;&emsp;select nfds=0 readfs={1,socket} writefds={0,0} exceptfds={0,0}   
&emsp;&emsp;&emsp;&emsp;timeout{tv_sec=2 tv_usec=0}  
&emsp;&emsp;recvfrom len=8192 flag=0 from=02 00 00 35 04 02 02 01-00 00 00 00 00 00 00 00  
&emsp;&emsp;&emsp;&emsp;fromlen=sizeof(sockaddr) buf=  
&emsp;&emsp;&emsp;&emsp;b4 c3 81 80 00 01 00 04-00 00 00 00 03 77 77 77  .............www  
&emsp;&emsp;&emsp;&emsp;09 6d 69 63 72 6f 73 6f-66 74 03 63 6f 6d 00 00  .microsoft.com..  
&emsp;&emsp;&emsp;&emsp;01 00 01 c0 0c 00 05 00-01 00 00 0c fd 00 1a 06  ................  
 
                ntohs hostshort=13568
                shutdown how=SD_SEND
                closesocket

## 结果 
&emsp;&emsp;直接使用udp层链接即可，另外按照前面所述，用WinHex可得到ip列表，此时要解决的问题都已解决，可以完整地写一个程序了^_^ 
* 4.2.2.1,美国 科罗拉多州布隆菲尔德市Level 3通信公司
* 4.2.2.2,美国 科罗拉多州布隆菲尔德市Level 3通信公司
* 4.2.2.3,美国 科罗拉多州布隆菲尔德市Level 3通信公司
* 4.2.2.4,美国 科罗拉多州布隆菲尔德市Level 3通信公司
* 4.2.2.5,美国 科罗拉多州布隆菲尔德市Level 3通信公司

## 自制DNS加速器
&emsp;&emsp;1500行代码用boost和stl一气呵成，预期可以支持ipv6 dns，批量测试/停止，解析域名镜像（如google），支持大dns库和自定义dns库，甚至以后和代理功能结合到一起。由于是初稿，因此有bug，这里贴出源码，给出工程，望和大家一起探讨改进之处，理论上说这个工具会是很强大的！

```C++
//functions.h
#pragma once  
#ifndef FASTERDNS_COMMONHEADER  
#define FASTERDNS_COMMONHEADER  
#endif  
   
#include <string>  
#include <vector>  
#include <boost/asio.hpp>  
#include <boost/function.hpp>  
#include <boost/thread/mutex.hpp>  
#include <boost/thread.hpp>  
#include <boost/thread/detail/thread_group.hpp>  
#include <deque>  
#include <set>  
#include "sqlite3.h"  
   
#define DATABASE "dnss.db"  
#define CSVDATA "nameservers.csv"  
   
class testSpeedData//每轮测试所用数据  
{  
public:  
    testSpeedData(char* _ip, char* _location) :ip(_ip), location(_location)  
    {  
        responsetime = 0;  
        testtime = 0;  
        lefttime = 0;  
        failtime = 0;  
        timeout = 0;  
    }  
public:  
    std::string ip;//DNS地址  
    std::string location;//所在地  
    int responsetime;//总响应时间  
    int testtime;//总测试次数  
    int lefttime;//剩余次数  
    int failtime;//失败次数  
    int timeout;//超时时间  
};  
   
typedef int(*DBCALLBACK)(void *NotUsed, int argc, char **argv, char **azColName);  
typedef std::vector<std::string> spliter;  
typedef std::vector<std::pair<std::string,std::string>> dbData;  
   
//数据库相关  
bool doadddata(dbData* dataparam, HWND progctrl);  
bool dodeldata(std::vector<std::string>* dataparam, HWND progctrl);  
bool getcount(int& count);  
bool getdata(std::vector<testSpeedData>* dataparam, HWND listctrl, HWND statu, boost::mutex* mu);  
   
   
//网络操作相关  
bool dohttpconnect(const std::string& ip, uint32_t& timeout);  
bool dodnsconnect(const std::string& ip, uint32_t& timeout, std::vector<uint8_t>& bufferdata);  
void makedns(std::vector<uint8_t>& bufferdata, const std::string& testdomain);  
void replaceXid(std::vector<uint8_t>& bufferdata, uint16_t Xid);  
void resolveAnswer(std::vector<uint8_t>& bufferdata, std::set<std::string>& resolvedip, int originlen);  
   
//系统设置相关  
bool getinterface(std::vector<std::string>& interfaces);  
bool setinterface(std::vector<std::string>& interfaces, std::vector<std::string>& addresss, HWND progctrl);  
   
enum 
{  
    //解析CSV数据文件  
    CSV_IP = 0,  
    CSV_NAME,  
    CSV_COUNTRYID,  
    CSV_CITY,  
    CSV_MAX,  
    //解析普通数据文件  
    COM_IP = 0,  
    COM_LOCATION,  
    COM_MAX,  
    //端口  
    PORT_HTTP = 80,  
    PORT_DNS = 53,  
    //DNS包  
    SIZE_DNSHEADER = sizeof(DNS_HEADER),  
    SIZE_DNSTAILER = 5,  
    //值域  
    INIT_THREADNUM = 1,  
    LOW_THREADNUM = 0,  
    HIGH_THREADNUM = 1000,  
    INIT_TIMEOUT = 1000,  
    LOW_TIMEOUT = 10,  
    HIGH_TIMEOUT = 10000,  
    INIT_TESTNUM = 1,  
    LOW_TESTNUM = 0,  
    HIGH_TESTNUM = 100,  
    //显示列  
    COL_IP = 0,  
    COL_LOC,  
    COL_RES,//平均响应时间  
    COL_FAIL,//失败次数/总次数  
    //其他  
    IDC_PROGRESS = 12345,  
    MAX_DOMAIN_LENGTH = 256,  
    MAXS_RECVNUM = 65536,  
    HTTP_CONNECTTIME = 5000,  
    TIMER1_ID=10000,  
};  
   
class ipcontainer:public boost::noncopyable  
{  
public:  
    static ipcontainer& getInstance();  
    bool adddata(const std::string& ip, const std::string& addr);  
    bool deldata(const std::string& ip);  
    bool getdata(std::vector<testSpeedData>* dataparam,HWND listctrl);  
    bool getcount(int& pcount);  
    virtual ~ipcontainer();  
private:  
    ipcontainer();  
    static int getdatacallback(void *NotUsed, int argc, char **argv, char **azColName);  
    static int getcountcallback(void *NotUsed, int argc, char **argv, char **azColName);  
private:  
    sqlite3* db;  
    static int* pcount;  
    static std::vector<testSpeedData>* testdata;  
    static HWND pwnd;  
};  
   
template<typename D>  
class ThreadPool  
{  
public:  
    ThreadPool(boost::mutex& _mu) :mu(_mu)  
    {  
        threadnum = 0;  
        task = NULL;  
    }  
   
    virtual ~ThreadPool()  
    {  
        if (ppio)  
        {  
            for (int i = 0; i < threadnum; i++)  
                delete ppio[i];  
            delete[]ppio;  
        }  
    }  
   
    bool Attach(int _threadnum, std::deque<int>* _task, std::vector<D>* _dataarray, HWND _listview)  
    {  
        if (!_task || _task->empty() || !_dataarray)  
            return false;//任务未结束  
        task = _task;  
        dataarray = _dataarray;  
        threadnum = _threadnum;  
        listview = _listview;  
        ppio = new boost::asio::io_service*[threadnum];  
        if (!ppio)  
            return false;  
        for (int i = 0; i < threadnum; i++)  
        {  
            ppio[i] = new boost::asio::io_service;  
            if (!ppio[i])//模拟new[]的行为，释放之前成功分配的内存  
            {  
                for (int j = i-1; j >= 0; j--)  
                    delete ppio[j];  
                delete[]ppio;  
                return false;  
            }  
        }  
        return true;  
    }  
   
    bool Exec(void(*f)(int, std::vector<D>*, HWND))  
    {  
        try 
        {  
            boost::thread_group tg;  
            //建立线程组  
            for (int i = 0; i < threadnum; i++)  
            {  
                tg.create_thread(boost::bind(&ThreadPool::wrapper, this, ppio[i],f));  
            }  
            //等待所有任务执行完毕  
            tg.join_all();  
        }  
        catch (...)  
        {  
            //确保释放new  
            return false;  
        }  
        return true;  
    }  
   
private:  
    //用于将task里的任务分配各给threadnum个活动线程，分配时要进行线程同步  
    void wrapper(boost::asio::io_service* service, void(*f)(int, std::vector<D>*, HWND))  
    {  
        while (true)  
        {  
            mu.lock();  
            if (task->empty())  
            {  
                mu.unlock();  
                return;  
            }  
            service->post(boost::bind(f, task->front(), dataarray, listview));  
            task->pop_front();  
            mu.unlock();  
            service->run();  
            service->reset();  
        }  
    }  
   
private:  
    int threadnum;  
    HWND listview;  
    boost::mutex& mu;//分配任务用锁  
    std::deque<int>* task;//要分配的总任务索引  
    std::vector<D>* dataarray;//原始数据  
    boost::asio::io_service** ppio;//耗费的线程资源       
};  
   
#pragma pack(push)  
#pragma pack(2)  
typedef struct _DNSANSWER_HEADER  
{  
    WORD Name;  
    WORD Type;  
    WORD Class;  
    DWORD Timetolive;  
    WORD Datalength;  
}DNSANSWER_HEADER, *PDNSANSWER_HEADER;  
#pragma pack(pop)  
   
#define DNS_BYTE_FLIP_ANSWERHEADER_COUNTS(pHeader)\  
    {\  
    PDNSANSWER_HEADER _head = (pHeader); \  
    INLINE_HTONS(_head->Name,        _head->Name);\  
    INLINE_HTONS(_head->Type,        _head->Type); \  
    INLINE_HTONS(_head->Class,       _head->Class); \  
    INLINE_HTONS(_head->Timetolive, _head->Timetolive); \  
    INLINE_HTONS(_head->Datalength, _head->Datalength); \  
    }  
   
class udpclient//只为做超时处理  
{  
public:  
    udpclient(const boost::asio::ip::udp::endpoint& listen_endpoint) :socket_(io_service_, listen_endpoint), deadline_(io_service_)  
    {  
        deadline_.expires_at(boost::posix_time::pos_infin);  
        check_deadline();  
    }  
    std::size_t receive(const boost::asio::mutable_buffer& buffer, boost::posix_time::time_duration timeout, boost::system::error_code& ec)  
    {  
        deadline_.expires_from_now(timeout);  
        ec = boost::asio::error::would_block;  
        std::size_t length = 0;  
        socket_.async_receive(boost::asio::buffer(buffer), boost::bind(&udpclient::handle_receive, _1, _2, &ec, &length));  
        do io_service_.run_one(); while (ec == boost::asio::error::would_block);  
        return length;  
    }  
private:  
    void check_deadline()  
    {  
        if (deadline_.expires_at() <= boost::asio::deadline_timer::traits_type::now())  
        {  
            socket_.cancel();  
            deadline_.expires_at(boost::posix_time::pos_infin);  
        }  
        deadline_.async_wait(boost::bind(&udpclient::check_deadline, this));  
    }  
    static void handle_receive(const boost::system::error_code& ec, std::size_t length,  
        boost::system::error_code* out_ec, std::size_t* out_length)  
    {  
        *out_ec = ec;  
        *out_length = length;  
    }  
private:  
    boost::asio::io_service io_service_;  
    boost::asio::deadline_timer deadline_;  
public:  
    boost::asio::ip::udp::socket socket_;  
};  
   
class tcpclient//只为做超时处理  
{  
public:  
    tcpclient() :socket_(io_service_), deadline_(io_service_)  
    {  
        deadline_.expires_at(boost::posix_time::pos_infin);  
        check_deadline();  
    }  
    void connect(const boost::asio::ip::tcp::endpoint ep, boost::posix_time::time_duration timeout, boost::system::error_code& ec)  
    {  
        deadline_.expires_from_now(timeout);  
        ec = boost::asio::error::would_block;  
        socket_.async_connect(ep,boost::bind(&tcpclient::handle_connect, _1, &ec));  
        do io_service_.run_one(); while (ec == boost::asio::error::would_block);  
    }  
private:  
    void check_deadline()  
    {  
        if (deadline_.expires_at() <= boost::asio::deadline_timer::traits_type::now())  
        {  
            socket_.cancel();  
            deadline_.expires_at(boost::posix_time::pos_infin);  
        }  
        deadline_.async_wait(boost::bind(&tcpclient::check_deadline, this));  
    }  
    static void handle_connect(const boost::system::error_code& ec, boost::system::error_code* out_ec)  
    {  
        *out_ec = ec;  
    }  
private:  
    boost::asio::io_service io_service_;  
    boost::asio::deadline_timer deadline_;  
public:  
    boost::asio::ip::tcp::socket socket_;  
}; 


// FasterDNSDlg.cpp : 实现文件  
#include "stdafx.h"  
#include "FasterDNS.h"  
#include "FasterDNSDlg.h"  
#include "SetDnsDlg.h"  
#include "afxdialogex.h"  
#include <boost/scoped_ptr.hpp>  
#include <boost/bind.hpp>  
#include <boost/thread.hpp>  
#include <boost/range.hpp>  
#include <boost/algorithm/string.hpp>  
#include <boost/xpressive/xpressive.hpp>  
#include <boost/asio.hpp>  
#include <boost/lexical_cast.hpp>  
#include <boost/format.hpp>  
#include <queue>  
#include <fstream>  
#include <set>  
#include "functions.h"  
#include <stdlib.h>  
#include "ShowIp.h"  
#include "GlobalSetting.h"  
   
#ifdef _DEBUG  
#define new DEBUG_NEW  
#endif  
   
CFasterDNSDlg::CFasterDNSDlg(CWnd* pParent /*=NULL*/)  
    : CDialogEx(CFasterDNSDlg::IDD, pParent)  
{  
    m_hIcon = AfxGetApp()->LoadIcon(IDR_MAINFRAME);  
}  
   
void CFasterDNSDlg::DoDataExchange(CDataExchange* pDX)  
{  
    CDialogEx::DoDataExchange(pDX);  
}  
   
BEGIN_MESSAGE_MAP(CFasterDNSDlg, CDialogEx)  
    ON_WM_PAINT()  
    ON_BN_CLICKED(IDC_ADDDNS, &CFasterDNSDlg::OnBnClickedAddDNS)  
    ON_NOTIFY(LVN_GETDISPINFO, IDC_IPLIST, &CFasterDNSDlg::OnLvnGetdispinfoIplist)  
    ON_COMMAND(ID_ABOUT, &CFasterDNSDlg::OnAbout)  
    ON_COMMAND(ID_SETDNS, &CFasterDNSDlg::OnSetdns)  
    ON_NOTIFY(HDN_ITEMCLICK, 0, &CFasterDNSDlg::OnHdnItemclickIplist)  
    ON_NOTIFY(NM_RCLICK, IDC_IPLIST, &CFasterDNSDlg::OnNMRClickIplist)  
    ON_COMMAND(ID_DELDNS, &CFasterDNSDlg::OnDeldns)  
    ON_BN_CLICKED(IDC_ALLINONE, &CFasterDNSDlg::OnBnClickedAllinone)  
    ON_BN_CLICKED(IDC_START, &CFasterDNSDlg::OnBnClickedStart)  
    ON_BN_CLICKED(IDC_STOP, &CFasterDNSDlg::OnBnClickedStop)  
    ON_BN_CLICKED(IDC_VIEWIP, &CFasterDNSDlg::OnBnClickedViewip)  
    ON_WM_TIMER()  
    ON_COMMAND(ID_ADDDNS, &CFasterDNSDlg::OnAdddns)  
    ON_COMMAND(ID_MODIFY, &CFasterDNSDlg::OnModify)  
    ON_COMMAND(ID_STARTALL, &CFasterDNSDlg::OnStartall)  
    ON_COMMAND(ID_STOPALL, &CFasterDNSDlg::OnStopall)  
    ON_COMMAND(ID_TESTIP, &CFasterDNSDlg::OnTestip)  
    ON_COMMAND(ID_EXIT, &CFasterDNSDlg::OnExit)  
    ON_COMMAND(ID_ALLINONE, &CFasterDNSDlg::OnAllinone)  
    ON_COMMAND(ID_START, &CFasterDNSDlg::OnStart)  
    ON_COMMAND(ID_STOP, &CFasterDNSDlg::OnStop)  
END_MESSAGE_MAP()  
   
BOOL CFasterDNSDlg::OnInitDialog()  
{  
    CDialogEx::OnInitDialog();  
    SetIcon(m_hIcon, TRUE);         // 设置大图标  
    SetIcon(m_hIcon, FALSE);        // 设置小图标  
    m_list = (CListCtrl*)GetDlgItem(IDC_IPLIST);  
    m_list->SendMessage(LVM_SETEXTENDEDLISTVIEWSTYLE, LVS_EX_FULLROWSELECT, LVS_EX_FULLROWSELECT);  
    RECT rt;  
    m_list->GetClientRect(&rt);  
    m_list->InsertColumn(COL_IP, "DNS地址", LVCFMT_LEFT, rt.right / 5);  
    m_list->InsertColumn(COL_LOC, "所在地", LVCFMT_LEFT, rt.right / 5);  
    m_list->InsertColumn(COL_RES, "响应时间", LVCFMT_LEFT, rt.right / 5);  
    m_list->InsertColumn(COL_FAIL, "失败率", LVCFMT_LEFT, rt.right / 5);  
   
    boost::thread(boost::bind(getdata, &dataarray, m_list->m_hWnd, GetDlgItem(IDC_STATU)->m_hWnd, &datamutex));  
    issorting = false;  
    maxtask = 0;  
    repeattime = 3;  
    threadnum = 3;  
    timeout = 1000;  
    SetDlgItemText(IDC_DOMAIN, "www.microsoft.com");  
    SetTimer(TIMER1_ID, 1000, NULL);  
    return TRUE;  // 除非将焦点设置到控件，否则返回 TRUE  
}  
   
void CFasterDNSDlg::OnPaint()  
{  
    if (IsIconic())  
    {  
        CPaintDC dc(this); // 用于绘制的设备上下文  
        SendMessage(WM_ICONERASEBKGND, reinterpret_cast<WPARAM>(dc.GetSafeHdc()), 0);  
        int cxIcon = GetSystemMetrics(SM_CXICON);  
        int cyIcon = GetSystemMetrics(SM_CYICON);  
        CRect rect;  
        GetClientRect(&rect);  
        int x = (rect.Width() - cxIcon + 1) / 2;  
        int y = (rect.Height() - cyIcon + 1) / 2;  
        dc.DrawIcon(x, y, m_hIcon);  
    }  
    else 
    {  
        CDialogEx::OnPaint();  
    }  
}  
   
void CFasterDNSDlg::OnBnClickedAddDNS()  
{  
    CFileDialog  dlg(TRUE, NULL, NULL, OFN_HIDEREADONLY | OFN_OVERWRITEPROMPT, "*.*", NULL);  
    if (IDOK == dlg.DoModal())  
    {  
        //读取文件，解析出ip  
        std::fstream file;  
        std::string line;  
        dbData toadd;  
        boost::asio::ip::address addr;  
        spliter splitvec;  
        //获取大小  
        file.open(dlg.GetPathName(), std::ios::in);  
        file.seekg(0,std::ios::end);  
        int totalsize = file.tellg();  
        file.seekg(0,std::ios::beg);  
   
        CProgressCtrl m_progctrl;  
        m_progctrl.Create(WS_CHILD | WS_VISIBLE, CRect(0, 0, 400, 50), this,  
            IDC_PROGRESS);  
        m_progctrl.SetRange(0, 100);  
        boost::mutex::scoped_lock mdata(datamutex);  
   
        if (dlg.GetFileName() == CSVDATA)  
        {  
            getline(file, line);//跳过第一行  
            while (!file.eof())  
            {  
                int pos2 = file.tellg();  
                m_progctrl.SetPos(int(100*(double)file.tellg()/totalsize));  
                getline(file, line);  
                boost::split(splitvec, line, boost::is_any_of(","));  
                try 
                {  
                    if (splitvec.size() < CSV_MAX)  
                        throw "Error";  
                    addr = addr.from_string(splitvec[CSV_IP]);  
                }  
                catch (...)  
                {  
                    break;  
                }  
                LVITEM item = { 0 };  
                item.mask = LVIF_TEXT;  
                item.iItem = dataarray.size();  
                item.lParam = item.iItem;  
                item.pszText = LPSTR_TEXTCALLBACK;  
                dataarray.push_back(testSpeedData((char*)splitvec[CSV_IP].c_str(), (char*)(splitvec[CSV_COUNTRYID] + " " + splitvec[CSV_CITY]).c_str()));  
                m_list->InsertItem(&item);  
                toadd.push_back(make_pair(splitvec[CSV_IP], splitvec[CSV_COUNTRYID] + " " + splitvec[CSV_CITY]));  
            }  
        }  
        else 
        {//普通文件，ip地址和位置信息用逗号分隔  
            while (!file.eof())  
            {  
                m_progctrl.SetPos(int(100*(double)file.tellg() / totalsize));  
                getline(file, line);  
                boost::split(splitvec, line, boost::is_any_of(","));  
                try 
                {  
                    if (splitvec.size() < COM_MAX)  
                        throw "Error";  
                    addr = addr.from_string(splitvec[COM_IP]);  
                }  
                catch (...)  
                {  
                    break;  
                }  
                LVITEM item = { 0 };  
                item.mask = LVIF_TEXT;  
                item.iItem = dataarray.size();  
                item.lParam = item.iItem;  
                item.pszText = LPSTR_TEXTCALLBACK;  
                dataarray.push_back(testSpeedData((char*)splitvec[COM_IP].c_str(), (char*)splitvec[COM_LOCATION].c_str()));  
                m_list->InsertItem(&item);  
                toadd.push_back(make_pair(splitvec[COM_IP], splitvec[COM_LOCATION]));  
            }  
        }  
        file.close();  
        ProgressThread("正在添加",doadddata,&toadd);  
    }  
}  
   
template<typename F,typename D>  
void CFasterDNSDlg::ProgressThread(LPCTSTR wndname,F& threadfunc,D paramdata)  
{  
    CProgressCtrl m_progctrl;  
    m_progctrl.Create(WS_CHILD | WS_VISIBLE | PBS_SMOOTH, CRect(0, 0, 400, 50), this, IDC_PROGRESS);  
    m_progctrl.SetRange(0, 100);  
    m_progctrl.SetWindowText(wndname);  
    //绑定进度条到threadfunc，抽象出逻辑使在其内部动态更新进度条  
    boost::packaged_task<bool> pt(boost::bind(threadfunc, paramdata, m_progctrl.m_hWnd));  
    boost::unique_future<bool> uf = pt.get_future();  
    boost::thread(boost::move(pt));  
    uf.wait();  
    if (uf.get())  
        SetDlgItemText(IDC_STATU, "操作成功！");  
    else 
        SetDlgItemText(IDC_STATU, "操作失败！");  
}  
   
char toformat[256];  
   
void CFasterDNSDlg::OnLvnGetdispinfoIplist(NMHDR *pNMHDR, LRESULT *pResult)  
{//刷新数据  
    NMLVDISPINFO *pDispInfo = reinterpret_cast<NMLVDISPINFO*>(pNMHDR);  
    // TODO:  在此添加控件通知处理程序代码  
    int index = pDispInfo->item.iItem;  
    if (dataarray.size() == 0 || index > dataarray.size() - 1)  
        return;  
    testSpeedData& cur = dataarray[index];  
    switch (pDispInfo->item.iSubItem)  
    {  
    case COL_IP:  
        pDispInfo->item.pszText = (char*)cur.ip.c_str();  
        break;  
    case COL_LOC:  
        pDispInfo->item.pszText = (char*)cur.location.c_str();  
        break;  
    case COL_RES:  
        if (cur.responsetime == 0)  
            pDispInfo->item.pszText = "未测速";  
        else if (cur.testtime == cur.failtime)  
            pDispInfo->item.pszText = "测速失败";  
        else 
        {  
            sprintf_s(toformat, "%d ms", cur.responsetime / (cur.testtime - cur.failtime));  
            pDispInfo->item.pszText = toformat;  
        }  
        break;  
    case COL_FAIL:  
        if (dataarray[index].testtime == 0)  
            pDispInfo->item.pszText = "未测速";  
        else 
        {  
            sprintf_s(toformat, "%d/%d", cur.failtime, cur.testtime);  
            pDispInfo->item.pszText = toformat;  
        }  
        break;  
    }  
    *pResult = 0;  
}  
   
void CFasterDNSDlg::OnAbout()  
{  
    AfxMessageBox("该程序在‘彗星DNS优化器’基础上修改得到，增加了数据库，有望未来支持ipv6\r\n作者:lichao890427 qq:571652571");  
}  
   
void CFasterDNSDlg::OnSetdns()  
{//显示网卡窗口  
    std::vector<std::string> selip;  
    int uSelectedCount = m_list->GetSelectedCount();  
    int nItem = -1;  
    if (uSelectedCount > 0)  
    {  
        for (int i = 0; i < uSelectedCount; i++)  
        {  
            nItem = m_list->GetNextItem(nItem, LVNI_SELECTED);  
            selip.push_back(dataarray[nItem].ip);  
        }  
        boost::thread(boost::bind(&CFasterDNSDlg::DoDNSConnection, this));  
    }  
   
    CSetDnsDlg setdns(selip);  
    setdns.DoModal();  
}  
   
bool CompareIP(testSpeedData& lParam1, testSpeedData& lParam2)  
{  
    using namespace boost::asio;  
    ip::address& addr1 = ip::address::from_string(lParam1.ip);  
    ip::address& addr2 = ip::address::from_string(lParam2.ip);  
    if (addr1.is_v4())  
    {  
        if (addr2.is_v4())  
        {  
            return addr1.to_v4().to_ulong() < addr2.to_v4().to_ulong();  
        }  
        else 
            return true;  
    }  
    else 
    {  
        return false;  
    }  
}  
   
bool CompareLOC(testSpeedData& lParam1, testSpeedData& lParam2)  
{  
    return lParam1.location < lParam2.location;  
}  
   
bool CompareRES(testSpeedData& lParam1, testSpeedData& lParam2)  
{  
    if (lParam1.failtime || lParam1.lefttime)  
        return true;  
    else if (lParam2.failtime || lParam2.lefttime)  
        return false;  
    else if (!lParam1.testtime || !lParam2.testtime)  
        return lParam1.testtime < lParam2.testtime;  
    else 
        return lParam1.responsetime / lParam1.testtime < lParam2.responsetime / lParam2.testtime;  
}  
   
void CFasterDNSDlg::DoSort(bool(*f)(testSpeedData&, testSpeedData&))  
{  
    if (issorting)  
    {  
        AfxMessageBox("排序尚未完成！");  
        return;  
    }  
   
    boost::mutex::scoped_lock mdata(datamutex);  
    SetDlgItemText(IDC_STATU, "正在排序");  
    issorting = true;  
    std::sort(dataarray.begin(), dataarray.end(), f);  
    for (int i = 0; i < dataarray.size(); i++)  
    {  
        LVITEM item = { 0 };  
        item.mask = LVIF_TEXT;  
        item.iItem = i;  
        item.pszText = LPSTR_TEXTCALLBACK;  
        m_list->SendMessage(LVM_SETITEMTEXT, i, (LPARAM)&item);  
    }  
    SetDlgItemText(IDC_STATU, "排序完毕");  
    issorting = false;  
}  
   
void CFasterDNSDlg::OnHdnItemclickIplist(NMHDR *pNMHDR, LRESULT *pResult)  
{//排序  
    LPNMHEADER phdr = reinterpret_cast<LPNMHEADER>(pNMHDR);  
   
    switch (phdr->iItem)  
    {//不要直接修改listview，而是先修改和listview绑定的data，再通知listview  
        case COL_IP:  
            boost::thread(boost::bind(&CFasterDNSDlg::DoSort, this, CompareIP));  
            break;  
        case COL_LOC:  
            boost::thread(boost::bind(&CFasterDNSDlg::DoSort, this, CompareLOC));  
            break;  
        case COL_RES:  
            boost::thread(boost::bind(&CFasterDNSDlg::DoSort, this, CompareRES));  
            break;  
        case COL_FAIL:  
            break;  
    }  
    *pResult = 0;  
}  
   
void CFasterDNSDlg::OnNMRClickIplist(NMHDR *pNMHDR, LRESULT *pResult)  
{//右键弹出菜单  
    LPNMITEMACTIVATE pNMItemActivate = reinterpret_cast<LPNMITEMACTIVATE>(pNMHDR);  
    CMenu menu;  
    menu.LoadMenu(IDR_POPUPMENU);  
    CMenu* pPopup = menu.GetSubMenu(0);  
    POINT pt;  
    GetCursorPos(&pt);  
    pPopup->TrackPopupMenu(TPM_LEFTALIGN | TPM_LEFTBUTTON, pt.x, pt.y, this);  
    *pResult = 0;  
}  
   
void CFasterDNSDlg::OnDeldns()  
{//删除dns，先从数据库中删，再重新加载数据  
    int uSelectedCount = m_list->GetSelectedCount();  
    int nItem = -1;  
    if (uSelectedCount > 0)  
    {  
        boost::mutex::scoped_lock mdata(datamutex);  
        std::vector<std::string> temp;  
        for (int i = 0; i < uSelectedCount; i++)  
        {  
            nItem = m_list->GetNextItem(nItem, LVNI_SELECTED);  
            temp.push_back(dataarray[nItem].ip);  
        }  
        dodeldata(&temp, m_list->m_hWnd);  
        dataarray.clear();  
        m_list->DeleteAllItems();  
        boost::thread(boost::bind(getdata, &dataarray, m_list->m_hWnd, GetDlgItem(IDC_STATU)->m_hWnd, &datamutex));  
    }  
}  
   
std::set<std::string> resolvedip;  
std::string domain="www.microsoft.com";  
   
void testFunc(int index, std::vector<testSpeedData>* rawdata, HWND listview)  
{  
    testSpeedData& curdata = (*rawdata).at(index);  
    std::vector<uint8_t> bufferdata;  
    bool got = false;  
    while (curdata.lefttime)  
    {  
        makedns(bufferdata, domain);  
        int originlen = bufferdata.size();//发送包大小，用于定位返回answer  
        replaceXid(bufferdata, rand() & 0xFFFF);  
        uint32_t usedtime = curdata.timeout;  
        if (!dodnsconnect(curdata.ip, usedtime, bufferdata))  
            curdata.failtime++;  
        else 
        {  
            curdata.responsetime += usedtime;  
            if (!got)  
            {//解析返回包  
                resolveAnswer(bufferdata, resolvedip, originlen);  
                got = true;  
            }  
        }     
        curdata.lefttime--;  
    }  
    //刷新listview  
    LVITEM item = { 0 };  
    item.mask = LVIF_TEXT;  
    item.iItem = index;  
    item.pszText = LPSTR_TEXTCALLBACK;  
    SendMessage(listview, LVM_SETITEMTEXT, index, (LPARAM)&item);  
}  
   
void CFasterDNSDlg::OnTimer(UINT_PTR nIDEvent)  
{  
    char status[256];  
    sprintf_s(status,"%06d/%06d/%06d", task.size(), maxtask, dataarray.size());  
    SetDlgItemText(IDC_TESTPROG, status);  
    CDialogEx::OnTimer(nIDEvent);  
}  
   
void CFasterDNSDlg::DoDNSConnection()  
{//设置testSpeedData  
    maxtask = task.size();  
    CString domaina;  
    GetDlgItemText(IDC_DOMAIN, domaina);  
    ::domain = (LPCTSTR)domaina;  
    resolvedip.clear();  
    ThreadPool<testSpeedData> newtaskset(taskmutex);  
    newtaskset.Attach(threadnum, &task, &dataarray,m_list->m_hWnd);  
    newtaskset.Exec(testFunc);  
    OnBnClickedViewip();  
}  
   
void CFasterDNSDlg::InitOne(testSpeedData& obj)  
{//初始化单个数据  
    obj.failtime = 0;  
    obj.lefttime = repeattime;  
    obj.responsetime = 0;  
    obj.testtime = repeattime;  
    obj.timeout = timeout;  
}  
   
void CFasterDNSDlg::OnBnClickedAllinone()  
{  
    boost::mutex::scoped_lock mtask(taskmutex);  
    //list和data绑定，数据更改通过data更新list而不直接操作list，因此该锁同时绑定data和list  
    if (!task.empty())  
    {  
        AfxMessageBox("测试尚未结束，请先停止测试");  
        return;  
    }  
    int nCount = m_list->GetItemCount();  
    task.clear();  
    for (int i = 0; i < nCount; i++)  
    {  
        task.push_back(i);  
    }  
    boost::thread(boost::bind(&CFasterDNSDlg::DoDNSConnection, this));  
}  
   
void CFasterDNSDlg::OnBnClickedStart()  
{//测试选中  
    boost::mutex::scoped_lock mtask(taskmutex);  
    if (!task.empty())  
    {  
        AfxMessageBox("测试尚未结束，请先停止测试");  
        return;  
    }  
    int uSelectedCount = m_list->GetSelectedCount();  
    int nItem = -1;  
    if (uSelectedCount > 0)  
    {  
        task.clear();  
        for (int i = 0; i < uSelectedCount; i++)  
        {  
            nItem = m_list->GetNextItem(nItem, LVNI_SELECTED);  
            InitOne(dataarray[nItem]);  
            task.push_back(nItem);  
        }  
        boost::thread(boost::bind(&CFasterDNSDlg::DoDNSConnection, this));  
    }  
}  
   
void CFasterDNSDlg::OnBnClickedStop()  
{  
    //停止当前任务意味着清除剩余分配任务task  
    boost::mutex::scoped_lock mio(taskmutex);  
    //使用同一个锁保证不和分配任务线程操作冲突  
    task.clear();  
}  
   
void CFasterDNSDlg::OnBnClickedViewip()  
{  
    CShowIp ipdlg(domain, resolvedip);  
    ipdlg.DoModal();  
}  
   
void CFasterDNSDlg::OnAdddns()  
{  
    OnBnClickedAddDNS();  
}  
   
void CFasterDNSDlg::OnModify()  
{  
    CGlobalSetting settings(threadnum, timeout, repeattime);  
    settings.DoModal();  
}  
   
void CFasterDNSDlg::OnStartall()  
{  
    OnBnClickedAllinone();  
}  
   
void CFasterDNSDlg::OnStopall()  
{  
    OnBnClickedStop();  
}  
   
void CFasterDNSDlg::OnTestip()  
{  
    OnBnClickedViewip();  
}  
   
void CFasterDNSDlg::OnExit()  
{  
    SendMessage(WM_CLOSE, 0, 0);  
}  
   
void CFasterDNSDlg::OnAllinone()  
{  
    OnBnClickedAllinone();  
}  
   
void CFasterDNSDlg::OnStart()  
{  
    OnBnClickedStart();  
}  
   
void CFasterDNSDlg::OnStop()  
{  
    OnBnClickedStop();  
} 

// functions.cpp
#include "stdafx.h"    
#include <boost/function.hpp>  
#include <boost/asio.hpp>  
#include <boost/thread.hpp>  
#include <boost/bind.hpp>  
#include <boost/date_time.hpp>  
#include <boost/algorithm/string.hpp>  
#include <boost/lexical_cast.hpp>  
#include <boost/format.hpp>  
#include <boost/system/error_code.hpp>  
#include <vector>  
#include <set>  
#include <winsock.h>  
#include <stdint.h>  
#include <iphlpapi.h>  
#include <windows.h>  
#include "functions.h"  
#include "sqlite3.h"  
#pragma comment(lib,"sqlite3.lib")  
#pragma comment(lib,"iphlpapi.lib")  
/************************************************************************/ 
/* 单例模式获取实例                                                     */ 
/************************************************************************/ 
std::vector<testSpeedData>* ipcontainer::testdata = NULL;  
int* ipcontainer::pcount = NULL;  
HWND ipcontainer::pwnd = NULL;  
   
ipcontainer& ipcontainer::getInstance()  
{  
    boost::mutex mio;  
    boost::mutex::scoped_lock lock(mio);  
    static ipcontainer container;  
    return container;  
}  
   
/************************************************************************/ 
/* 向数据库中增加dns数据  
    ip:待添加dns服务器ip地址(非域名)   addr:待添加dns服务器地理位置  */ 
/************************************************************************/ 
bool ipcontainer::adddata(const std::string& ip, const std::string& addr)  
{  
    int status = SQLITE_OK;  
    char* zErrMsg = NULL;  
    std::string toadd = "insert into dnss values ('" + ip + "','" + addr + "')";  
    try 
    {  
        status = sqlite3_exec(db, toadd.c_str(), NULL, NULL, &zErrMsg);  
        if (status != SQLITE_OK)  
            throw "Add failed!";  
    }  
    catch (...)  
    {  
        return false;  
    }  
    return true;  
}  
   
/************************************************************************/ 
/* 从数据库中删除dns数据  
    ip:待删除dns服务器ip地址(非域名)                                   */ 
/************************************************************************/ 
bool ipcontainer::deldata(const std::string& ip)  
{  
    int status = SQLITE_OK;  
    char* zErrMsg = NULL;  
    std::string toadd = "delete from dnss where ipaddr = '" + ip + "'";  
    try 
    {  
        status = sqlite3_exec(db, toadd.c_str(), NULL, NULL, &zErrMsg);  
        if (status != SQLITE_OK)  
            throw "Del failed!";  
    }  
    catch (...)  
    {  
        return false;  
    }  
    return true;  
}  
   
/************************************************************************/ 
/* 获取dns数据总入口  
    dataparam:存放获得数据结果                                          */ 
/************************************************************************/ 
bool ipcontainer::getdata(std::vector<testSpeedData>* dataparam, HWND listctrl)  
{  
    try 
    {  
        ipcontainer::testdata = dataparam;  
        ipcontainer::pwnd = listctrl;  
        int status = SQLITE_OK;  
        char* zErrMsg = NULL;  
        status = sqlite3_exec(db, "select * from dnss", getdatacallback, NULL, &zErrMsg);  
        if (status != SQLITE_OK)  
            throw "Get failed!";  
    }  
    catch (...)  
    {  
        ipcontainer::testdata = NULL;  
        ipcontainer::pwnd = NULL;  
        return false;  
    }  
    ipcontainer::testdata = NULL;  
    ipcontainer::pwnd = NULL;  
    return true;  
}  
   
/************************************************************************/ 
/* 获取dns数据记录总数                                                  */ 
/************************************************************************/ 
bool ipcontainer::getcount(int& count)  
{  
    try 
    {  
        ipcontainer::pcount = &count;  
        int status = SQLITE_OK;  
        char* zErrMsg = NULL;  
        status = sqlite3_exec(db, "select count(*) from dnss", getcountcallback, NULL, &zErrMsg);  
        if (status != SQLITE_OK)  
            throw "Get failed!";  
    }  
    catch (...)  
    {  
        ipcontainer::pcount = NULL;  
        return false;  
    }  
    ipcontainer::pcount = NULL;  
    return true;  
}  
   
/************************************************************************/ 
/* 析构时关闭数据库                                                     */ 
/************************************************************************/ 
ipcontainer::~ipcontainer()  
{  
    ipcontainer::testdata = NULL;  
    ipcontainer::pcount = NULL;  
    ipcontainer::pwnd = NULL;  
    sqlite3_close(db);  
    db = NULL;  
}  
   
/************************************************************************/ 
/* 构造函数打开数据库                                                    */ 
/************************************************************************/ 
ipcontainer::ipcontainer()  
{  
    int status = SQLITE_OK;  
    db = NULL;  
    status = sqlite3_open(DATABASE, &db);  
    if (status != SQLITE_OK)  
        throw "Open failed!";  
}  
   
/************************************************************************/ 
/* 获取记录数据数组的回调函数，用于获取记录数据数组  
    argc:获得条目个数=2       argv[0]:ip          argv:location           */ 
/************************************************************************/ 
int ipcontainer::getdatacallback(void *NotUsed, int argc, char **argv, char **azColName)  
{  
    LVITEM item = { 0 };  
    item.mask = LVIF_TEXT;  
    item.iItem = ipcontainer::testdata->size();  
    item.lParam = item.iItem;  
    item.pszText = LPSTR_TEXTCALLBACK;  
    ipcontainer::testdata->push_back(testSpeedData(argv[0], argv[1]));  
    SendMessage(pwnd, LVM_INSERTITEM, 0, (LPARAM)&item);  
    return 0;  
}  
   
/************************************************************************/ 
/* 获取记录条目的回调函数，用于得到记录条目      
    argc:获得条目个数=1       argv[0]:记录条目                            */ 
/************************************************************************/ 
int ipcontainer::getcountcallback(void *NotUsed, int argc, char **argv, char **azColName)  
{  
    try 
    {  
        *ipcontainer::pcount = boost::lexical_cast<int>(argv[0]);  
    }  
    catch (...)  
    {  
        *ipcontainer::pcount = 0;  
    }  
    return 0;  
}  
   
/************************************************************************/ 
/* 增加dns数据总入口  
setpos:设置进度显示回调函数       dataparam:要添加的数据阵列*/ 
/************************************************************************/ 
bool doadddata(dbData* dataparam, HWND progctrl)  
{  
    int size = dataparam->size() / 100;  
    int i = 0, j = 0;  
    SendMessage(progctrl, PBM_SETPOS, 0, 0);  
    try 
    {  
        dbData::const_iterator itor = dataparam->begin();  
        ipcontainer& container=ipcontainer::getInstance();  
        while (itor != dataparam->end())  
        {  
            container.adddata(itor->first, itor->second);  
            ++itor;  
            ++i;  
            if (i >= size)  
            {  
                i = 0;  
                SendMessage(progctrl, PBM_SETPOS, ++j, 0);  
            }  
        }  
    }  
    catch (...)  
    {  
        //发生错误  
        return false;  
    }  
    return true;  
}  
   
/************************************************************************/ 
/* 删除dns数据总入口  
setpos:设置进度显示回调函数       dataparam:要删除的数据阵列*/ 
/************************************************************************/ 
bool dodeldata(std::vector<std::string>* dataparam, HWND progctrl)  
{  
    int size = dataparam->size() / 100;  
    int i = 0, j = 0;  
    SendMessage(progctrl, PBM_SETPOS, 0, 0);  
    try 
    {  
        std::vector<std::string>::iterator itor = dataparam->begin();  
        ipcontainer& container = ipcontainer::getInstance();  
        while (itor != dataparam->end())  
        {  
            container.deldata(*itor);  
            ++itor;  
            ++i;  
            if (i >= size)  
            {  
                i = 0;  
                SendMessage(progctrl, PBM_SETPOS, ++j, 0);  
            }  
        }  
    }  
    catch (...)  
    {  
        //发生错误  
        return false;  
    }  
    return true;  
}  
   
/************************************************************************/ 
/* 尝试http连接  
    ip:要测试连通性的地址        timeout:超时返回                        */ 
/************************************************************************/ 
bool dohttpconnect(const std::string& ip,uint32_t& timeout)  
{//ip不能是域名 timeout同时用于设置超时和计时  
    using namespace boost::asio;  
    using namespace boost::posix_time;  
    try 
    {  
        tcpclient c;  
        boost::system::error_code ec;  
        setsockopt(c.socket_.native(), SOL_SOCKET, SO_RCVTIMEO, (const char*)timeout, sizeof(timeout));  
        ip::tcp::endpoint ep(ip::address::from_string(ip), PORT_HTTP);  
        ptime ptStart = microsec_clock::local_time();  
        c.connect(ep, boost::posix_time::millisec(timeout), ec);  
        if (ec.value())  
            throw "error";  
        ptime ptEnd = microsec_clock::local_time();  
        timeout = (int32_t)(ptEnd - ptStart).total_milliseconds();  
    }  
    catch (...)  
    {  
        return false;  
    }  
    return true;  
}  
   
/************************************************************************/ 
/* 尝试dns连接  
    ip:要测试连通性的地址        timeout:超时返回          
    bufferdata:用于发送和接收数据                                        */ 
/************************************************************************/ 
bool dodnsconnect(const std::string& ip, uint32_t& timeout,std::vector<uint8_t>& bufferdata)  
{//bufferdata用于输入和输出 ip为服务器地址  
    using namespace boost::asio;  
    using namespace boost::posix_time;  
    try 
    {  
        udpclient c(ip::udp::endpoint(ip::udp::v4(), rand() & 0x7FFF + 0x8000));  
        boost::system::error_code ec;  
        setsockopt(c.socket_.native(), SOL_SOCKET, SO_RCVTIMEO, (const char*)timeout, sizeof(timeout));  
        setsockopt(c.socket_.native(), SOL_SOCKET, SO_BROADCAST, "\0\0\0\0", 4);  
        ip::udp::endpoint ep(ip::address::from_string(ip), PORT_DNS);  
        c.socket_.send_to(buffer((bufferdata)),ep);  
        bufferdata.clear();  
        bufferdata.resize(MAXS_RECVNUM);  
        ptime ptStart = microsec_clock::local_time();  
        c.receive(buffer(bufferdata), boost::posix_time::millisec(timeout), ec);  
        if (ec.value())  
            throw "error";  
        ptime ptEnd = microsec_clock::local_time();  
        timeout = (int32_t)(ptEnd - ptStart).total_milliseconds();  
    }  
    catch (...)  
    {  
        return false;  
    }  
    return true;  
}  
   
/************************************************************************/ 
/* 构造dns数据包  
    bufferdata:用于存储构造数据包结果  
    testdomain:用于验证的域名，不能用ip地址                              */ 
/************************************************************************/ 
void makedns(std::vector<uint8_t>& bufferdata, const std::string& testdomain)  
{//testip为验证ip，此函数用于testip更换时；Xid更换时用replaceXid  
    bufferdata.resize(sizeof(DNS_HEADER));  
    DNS_HEADER* pheader = (DNS_HEADER*)bufferdata.data();  
    memset(pheader, 0, sizeof(DNS_HEADER));  
    pheader->RecursionDesired = 1;  
    pheader->QuestionCount = 1;//查询一个域名  
    DNS_BYTE_FLIP_HEADER_COUNTS(pheader);  
    boost::asio::ip::address addr;  
    spliter splitvec;  
    boost::split(splitvec, testdomain, boost::is_any_of("."));  
    for (spliter::iterator itor = splitvec.begin(); itor != splitvec.end(); ++itor)  
    {  
        bufferdata.push_back((*itor).length());  
        bufferdata.insert(bufferdata.end(), (*itor).begin(),(*itor).end());  
    }  
    uint8_t tailer[SIZE_DNSTAILER] = { 0x00, 0x00, 0x01, 0x00, 0x01 };  
    bufferdata.insert(bufferdata.end(), tailer, tailer+SIZE_DNSTAILER);  
}  
   
/************************************************************************/ 
/* 修改dns数据包的TransactionID，考虑优化使用  
    bufferdata:待修改数据        Xid:新的TransactionID             */ 
/************************************************************************/ 
void replaceXid(std::vector<uint8_t>& bufferdata, uint16_t Xid)  
{  
    ((DNS_HEADER*)bufferdata.data())->Xid = htons(Xid);  
}  
   
/************************************************************************/ 
/*从返回包结果中解析获取到的地址     
    bufferdata:服务器返回的待解析数据  resolveip:存储结果  
    originlen:发送包大小*/ 
/************************************************************************/ 
void resolveAnswer(std::vector<uint8_t>& bufferdata, std::set<std::string>& resolvedip, int originlen)  
{  
    DNS_HEADER* pheader = (DNS_HEADER*)bufferdata.data();  
    DNS_BYTE_FLIP_HEADER_COUNTS(pheader);  
    uint8_t* ptr = (uint8_t*)pheader + originlen;  
    for (int i = 0; i < pheader->AnswerCount; i++)  
    {  
        DNSANSWER_HEADER* aheader = (DNSANSWER_HEADER*)ptr;  
        DNS_BYTE_FLIP_ANSWERHEADER_COUNTS(aheader);  
        if(aheader->Type == 1)//A:host address  
        {  
            BYTE* cdata = (BYTE*)aheader + sizeof(DNSANSWER_HEADER);  
            if (aheader->Datalength == 4)//ipv4  
            {  
                char temp[16];  
                sprintf_s(temp, "%d.%d.%d.%d", cdata[0], cdata[1], cdata[2], cdata[3]);  
                resolvedip.insert(std::string(temp));  
            }  
            else if (aheader->Datalength == 6)//ipv6  
            {  
                //v6的没有条件研究呢。。。  
            }  
        }  
        ptr += aheader->Datalength + sizeof(DNSANSWER_HEADER);  
    }  
}  
   
/************************************************************************/ 
/* 获取网卡适配器  
    interfaces:存储结果                                                 */ 
/************************************************************************/ 
bool getinterface(std::vector<std::string>& interfaces)  
{  
    DWORD dwRetVal = 0;  
    ULONG outBufLen = 15000;  
    LPVOID lpMsgBuf = NULL;  
    PIP_ADAPTER_ADDRESSES pAddress = NULL;  
    interfaces.clear();  
    do 
    {  
        pAddress = (IP_ADAPTER_ADDRESSES*)HeapAlloc(GetProcessHeap(), 0, outBufLen);  
        if (pAddress == NULL)  
            return false;  
        dwRetVal = GetAdaptersAddresses(AF_UNSPEC, GAA_FLAG_INCLUDE_PREFIX, NULL, pAddress, &outBufLen);  
        if (dwRetVal == ERROR_BUFFER_OVERFLOW)  
        {  
            HeapFree(GetProcessHeap(), 0, pAddress);  
            pAddress = NULL;  
        }  
    } while (dwRetVal == ERROR_BUFFER_OVERFLOW);  
    if (dwRetVal != NO_ERROR)  
        return false;  
   
    PIP_ADAPTER_ADDRESSES pCurrAddresses = pAddress;  
    while (pCurrAddresses)  
    {  
        HKEY hkResult;  
        std::string query = "SYSTEM\\CurrentControlSet\\Control\\Network\\{4D36E972-E325-11CE-BFC1-08002BE10318}\\";  
        query += pCurrAddresses->AdapterName;  
        query += "\\Connection\\";  
        dwRetVal = RegOpenKey(HKEY_LOCAL_MACHINE, query.c_str(), &hkResult);  
        if (dwRetVal == ERROR_SUCCESS)  
        {  
            char buffer[256];  
            LONG cbData = 256;  
            dwRetVal = RegQueryValue(hkResult, "Name", buffer, &cbData);  
            if (dwRetVal == ERROR_SUCCESS)  
            {  
                interfaces.push_back((char*)buffer);  
            }  
        }  
        pCurrAddresses = pCurrAddresses->Next;  
    }  
   
    if (pAddress)  
        HeapFree(GetProcessHeap(), 0, pAddress);  
    return true;  
}  
   
/************************************************************************/ 
/* 设置指定网卡适配器dns  
    setpos:设置进度显示回调函数       interfaces:待设置适配器         
    address为点分十进制地址                                             */ 
/************************************************************************/ 
bool setinterface(std::vector<std::string>& interfaces, std::vector<std::string>& addresss, HWND progctrl)  
{  
    using namespace boost::asio;  
    STARTUPINFO StartupInfo = { 0 };  
    PROCESS_INFORMATION ProcessInfo = { 0 };  
    StartupInfo.cb = sizeof(STARTUPINFO);  
    StartupInfo.dwFlags = STARTF_USESHOWWINDOW;  
    StartupInfo.wShowWindow = SW_HIDE;  
    std::string cmd;  
    int count = 0;  
    SendMessage(progctrl, PBM_SETPOS, 0, 0);  
    for (std::vector<std::string>::iterator itori = interfaces.begin(); itori != interfaces.end(); ++itori)  
    {  
        BOOL bRet;  
        cmd = "netsh interface ipv4 set dns name=\"" + *itori + "\" source=static addr=none";  
        //清除DNS设置     
        bRet=CreateProcess(NULL, (char*)cmd.c_str(), NULL, NULL, FALSE, CREATE_NO_WINDOW, NULL, NULL, &StartupInfo, &ProcessInfo);  
        if (!bRet)  
            return false;  
        if (ProcessInfo.hProcess)  
            WaitForSingleObject(ProcessInfo.hProcess, INFINITE);  
        cmd = "netsh interface ipv6 set dns name=\"" + *itori + "\" source=static addr=none";  
        bRet = CreateProcess(NULL, (char*)cmd.c_str(), NULL, NULL, FALSE, CREATE_NO_WINDOW, NULL, NULL, &StartupInfo, &ProcessInfo);  
        if (!bRet)  
            return false;  
        if (ProcessInfo.hProcess)  
            WaitForSingleObject(ProcessInfo.hProcess, INFINITE);  
        SendMessage(progctrl, PBM_SETPOS, int(100 * (float)count++ / interfaces.size()), 0);  
        SetWindowText(progctrl, ("正在清除设置 " + *itori).c_str());  
        for (std::vector<std::string>::iterator itora = addresss.begin(); itora != interfaces.end(); ++itora)  
        {  
            if ((ip::address::from_string(*itora)).is_v4())  
                cmd = "netsh interface ipv4 add dns name=\"" + *itori + "\" addr=" + *itora;  
            else 
                cmd = "netsh interface ipv6 add dns name=\"" + *itori + "\" addr=" + *itora;  
            SetWindowText(progctrl, ("正在设置 " + *itora).c_str());  
            bRet = CreateProcess(NULL, (char*)cmd.c_str(), NULL, NULL, FALSE, CREATE_NO_WINDOW, NULL, NULL, &StartupInfo, &ProcessInfo);  
            if (!bRet)  
                return false;  
            if (ProcessInfo.hProcess)  
                WaitForSingleObject(ProcessInfo.hProcess, INFINITE);  
        }  
    }  
    return true;  
}  
   
/************************************************************************/ 
/* 封装getcount                                                         */ 
/************************************************************************/ 
bool getcount(int& count)  
{  
    try 
    {  
        ipcontainer::getInstance().getcount(count);  
    }  
    catch (...)  
    {  
        return false;  
    }  
    return true;  
}  
   
/************************************************************************/ 
/* 封装getdata                                                          */ 
/************************************************************************/ 
bool getdata(std::vector<testSpeedData>* dataparam, HWND listctrl, HWND statu, boost::mutex* mu)  
{  
    boost::mutex::scoped_lock mdata(*mu);  
    SetWindowText(statu, "正在初始化数据！");  
    try 
    {  
        ipcontainer::getInstance().getdata(dataparam, listctrl);  
    }  
    catch (...)  
    {  
        return false;  
    }  
    SetWindowText(statu, "初始化完毕！");  
    return true;  
}

```