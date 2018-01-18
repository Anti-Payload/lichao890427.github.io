---
layout: post
title: 关于IDA库函数识别技术的讨论
categories: Reverse_Engineering
description: 关于IDA库函数识别技术的讨论
keywords: IDA Reverse_Engineering
---

## 1.lib是否应该分类？
&emsp;&emsp;我并没有系统研究过lib文件格式，不过从某些解析器可以看出含有目录结构，是多个文件打包处理后形成的一种文件，其中的文件可以通过VS自带工具解包出来。一般含有2种子文件形式：.dll和.obj。前者是动态链接库编译生成的文件形式，里面仅有链接信息而没有实际执行代码，用于定位到dll中的函数，而obj则是代码编译产生的可执行机器码。对于静态链接库工程编译后得到的lib（我称为“第一种lib文件格式”），用ida打开可以看到这个lib包含多个obj文件，给第三方使用时只需要用#pragma comment(lib,"*.lib")包含进来即可，结果是obj中的代码转移到生成的可执行文件中；对于动态链接库工程编译后得到的lib（我称为“第一种lib文件格式”）和dll，lib用IDA打开可以发现包含多个dll文件，他们只有定位dll函数的作用，而实际执行代码在dll中，给第三方使用时，如果单纯用dll则调用LoadLibrary和GetProcAddress系列API获得函数指针从而进行调用，如果同时使用lib和dll则只需要用#pragma comment(lib,"*.lib")包含进来即可使用里面的函数，这两种方式我把他称为“静态方式的动态链接”和“动态方式的动态链接”，无论哪种方式，dll都必须存在于可执行文件目录，否则无法运行。  
&emsp;&emsp;对于“第二种lib文件格式”解包的dll文件分析其PE格式，可以发现并不是常规的dll格式也就是PE格式，而只是一部分，且含有.debug .idata段，目前还没做深入研究。

## 2.如何找到某种框架、运行库链接阶段用到的文件？
&emsp;&emsp;在前一个帖子http://www.0xaa55.com/forum.php? ... tid=1021&extra=《关于MSVC的几个问题的研究》中，我已经提供了使用procmon监视文件系统获取编译使用文件的方法，在这里我提供另一种方法——使用调试器，这种方法比较高级也比较彻底，不过需要对调试指令有所了解。使用WinDbg打开MSVC主程序msdev.exe，打开对话框底部勾选“调试子进程”，这样对于由于编译调用CreateProcess产生的一系列子进程及孙进程都可进行调试。载入后运行，在MSVC建立MFC工程，编译前在WinDbg中暂停该进程下断点：  
`bp Kernel32!CreateFileA "da poi([esp+4]);g";bp Kernel32!CreateFileW "du poi([esp+4]);g";g`  
&emsp;&emsp;之后Ctrl+F5编译，运行，此后在WinDbg中会产生异常暂停几次，这是由于产生了新进程，一旦遇到这种情况需要在新进程里重新设置以上断点，总共十几次，最后得到满满的记录，经过分析，使用到的文件确实多，调用编译程序的过程也一目了然，而mfc相关的文件自然在mfc文件夹下了，因此锁定*mfc*文件夹下的*.lib文件，可以找到vc98\mfc\lib\nafxcwd.lib。就这一个9M的lib？没错，不用怀疑，所有基础MFC的实现都在这里了，有兴趣的可以打开里面$UWD\strcore.obj看下，是CString的实现。该方法可以通用于其他库编译用文件的检测，例如ATL、第三方库等。

## 3.MFC静态编译是否用到mfc*.lib
&emsp;&emsp;最简单的方法是删掉这些文件并编译（记得备份啊^_^），发现没报错，说明没有用到。这些文件用于“静态方式的动态链接”（见1）。从问题2可知MFC采取静态链接方式编译的程序用到的库有nafxcwd.lib

## 4.问题：如何将.dll和.obj混合成.lib
&emsp;&emsp;经常看到有lib文件是动态链接库和静态链接库混合形式的，这里提供一种制作思路。首先考虑如何将.lib拆分，来看下lib命令（有兴趣的可以逆向分析lib.exe，可以发现等同于link.exe -lib命令），MSDN上有帮助：
```Txt
usage: LIB [options] [files]
   options:

      /DEF[:filename]                用DEF文件生成输入lib文件（“第二种lib文件格式”）和一个.exp  
      /EXPORT:symbol                        
      /EXTRACT:membername        解压指定文件
      /INCLUDE:symbol
      /LIBPATH:dir
      /LIST[:filename]                列出目录文件，和EXTRACT合用可以解包出文件
      /MACHINE:{AM33|ARM|EBC|IA64|M32R|MIPS|MIPS16|MIPSFPU|MIPSFPU16|MIPSR41XX|
                SH3|SH3DSP|SH4|SH5|THUMB|X86}
      /NAME:filename
      /NODEFAULTLIB[:library]
      /NOLOGO
      /OUT:filename
      /REMOVE:membername
      /SUBSYSTEM:{CONSOLE|EFI_APPLICATION|EFI_BOOT_SERVICE_DRIVER|
                  EFI_ROM|EFI_RUNTIME_DRIVER|NATIVE|POSIX|WINDOWS|
                  WINDOWSCE}[,#[.##]]
      /VERBOSE
```
实例：  
* lib /def:yourdll.def /machine:i386 /out:yourdll.lib
* lib /def:1.def /export:_fn3=fn2 /machine:i386 /out:2.lib 1.lib
* lib /list 1.lib => 1.obj 2.obj ... => lib /extract:2.obj 1.lib      => 从1.lib解压出2.obj
* lib 1.dll stdafx.obj   /out:ll.lib     => 合并1.dll stdafx.obj到ll.lib

## 5.ida如何制作sig文件？
ida的flair扩展插件提供了制作sig文件的功能，其目录下有：
* dumpsig.exe        将sig文件转储为txt文本
* plb.exe                从omf格式的库文件(.lib .o .obj 等)生成pat
* pcf.exe                从coff格式的库文件生成pat
* pelf.exe        从elf格式的库文件生成pat
* ppsx                从索尼PlayStation PSX格式的库文件生成pat
* ptmobj.exe        从TriMedia格式的库文件生成pat
* pomf166.exe        从Kiel OMF 166目标文件生成pat
* pmacho.exe        从Mach-O目标文件生成pat
* sigmake.exe        从pat生成sig
* zipsig.exe        压缩解压sig文件

&emsp;&emsp;对于静态库A.lib(A.o A.obj 等)文件，先判定A的二进制格式，分别选择plb pcf等转换成pat，若这一过程中提示格式不正确，则需要从lib中除去格式不同的那些obj，再操作。生成pat后使用sigmake生成sig，这一过程中若出现冲突，则需要修改exc中冲突的函数，一般规则为：在每组相互冲突的函数中，sigmake让你仅指定一个函数作为相关签名的匹配函数。任何时候，如果在数据库中发现一个对应的签名，并且你想应用一个函数的名称，那么，你可以在该函数名称前附加一个加号（+）；如果你只想在数据库中添加某个函数的注释，则在该函数名称前附加一个减号（-）；如果在数据库中发现对应的签名时，你不想应用任何名称，那么，你不需要添加任何符号。之后再使用sigmake即可。例：nafxcwd.lib  
`pcf nafxcwd.lib nafxcwd.pat`  
&emsp;&emsp;得到pat内容为：

```Txt
558BEC51894DFC8B45FC8B8094000000C1E81783E0018BE55DC3558BEC51894D 25 3FBB 05E0 :0000 ?IsOptimizedDraw@COleControl@@QAEHXZ :001A ?Is
8B4DF0E8........C3B8........E9........8B4DF0E8........C3B8...... 00 0000 004C :0000 ? ^0004 ??1CAsyncMonikerFile@@UAE@XZ ^000F ___
558BEC51894DFC8B4DFCE8........8B450883E00185C074098B4DFC51E8.... 00 0000 002B :0000 ??_GCDataPathProperty@@UAEPAXI@Z ^000B ??1CDat
558BEC6AFF68........64A1........50648925........51894DF0C745FC00 0A 4EBE 004B :0000 ??1CDataPathProperty@@UAE@XZ ^000C __except_li
8B4DF0E8........C3B8........E9.................................. 00 0000 0013 :0000 ? ^0004 ??1CAsyncMonikerFile@@UAE@XZ ^000F ___
558BEC51894DFC8B4DFCE8........8B450883E00185C074098B4DFC51E8.... 00 0000 002B :0000 ??_GCCachedDataPathProperty@@UAEPAXI@Z ^000B ?
558BEC6AFF68........64A1........50648925........51894DF0C745FC00 0A 2ED9 004B :0000 ??1CCachedDataPathProperty@@UAE@XZ ^000C __exc
8B4DF0E8........C3B8........E9.................................. 00 0000 0013 :0000 ? ^0004 ??1CDataPathProperty@@UAE@XZ ^000F ___
558BEC51894DFCB8........8BE55DC3558BEC83EC18566A2068........8B45 03 BFB8 06EC :0000 ?GetMessageMap@CStockPropPage@@MBEPBUAFX_MSGMA
8B4DF0E8........C3B8........E9.................................. 00 0000 0013 :0000 ? ^0004 ??1COlePropertyPage@@UAE@XZ ^000F ___C
558BEC51894DFC8B4DFCE8........8B450883E00185C074098B4DFC51E8.... 00 0000 002B :0000 ??_GCStockPropPage@@UAEPAXI@Z ^000B ??1CStockP
558BEC6AFF68........64A1........50648925........51894DF0C745FC00 0D A621 004E :0000 ??1CStockPropPage@@UAE@XZ ^000C __except_list 
8B4DF0E8........C3B8........E9.................................. 00 0000 0013 :0000 ? ^0004 ??1COlePropertyPage@@UAE@XZ ^000F ___C
558BEC51894DFCB8........8BE55DC3................................ 00 0000 0010 :0000 ?GetRuntimeClass@CStockPropPage@@UBEPAUCRuntim
558BEC6A108B450C508B4D0851E8........83C40CF7D81BC0405DC3........ 00 0000 001C :0000 ?IsEqualGUID@@YAHABU_GUID@@0@Z ^000E _memcmp 
558BEC6A008B4510508B4D0C518B550852E8........5DC20C00............ 00 0000 001A :0000 ?AtlW2AHelper@@YGPADPADPBGH@Z ^0012 ?AtlW2AHel
558BEC535657837D0C00751E68........6A006A3C68........6A02E8...... 00 0000 0088 :0000 ?AtlW2AHelper@@YGPADPADPBGHI@Z ^000D ??_C@_08N
558BEC51894DFCB8........8BE55DC3558BEC6AFF68........64A1........ 04 9F62 115D :0000 ?GetMessageMap@CPicturePropPage@@MBEPBUAFX_MSG
8B4DF0E8........C38B4DF081C1E4000000E8........C3B8........E9.... 00 0000 012A :0000 ? ^0004 ??1CStockPropPage@@UAE@XZ ^0013 ??1CCo
558BEC51894DFC8B4DFCE8........8B450883E00185C074098B4DFC51E8.... 00 0000 002B :0000 ??_GCPicturePropPage@@UAEPAXI@Z ^000B ??1CPict
558BEC6AFF68........64A1........50648925........51894DF0C745FC00 0D A621 004E :0000 ??1CStockPropPage@@UAE@XZ ^000C __except_list 
。。。。。。。。。。。。。
```
`sigmake nafxcwd.pat MFC.sig`  
&emsp;&emsp;这步执行后产生：
```Txt
I:\软件\ida61\sig>sigmake nafxcwd.pat MFC.sig
MFC.sig: modules/leaves: 1471/586, COLLISIONS: 27
See the documentation to learn how to resolve collisions.
```  
&emsp;&emsp;编辑MFC.exc解决冲突后，重新执行成功得到sig

## 6.为何ida官方未提供dll制作sig文件工具
&emsp;&emsp;一般不需要dll的二进制代码，而需要lib的二进制代码，假设有同样功能的lib和dll，其中lib是静态链接库工程生成而dll是链接库工程生成，第三方exe在使用时，lib所嵌入在exe中的模块经过重定位，其字节码可能不同于dll中同样的模块。这里说明我发现的几个事实：  
&emsp;&emsp;编译好的静态库lib在于exe链接时，是不会被编译器优化的，无论exe怎样配置，优化阶段仅在编译阶段，也就是说如果lib在编译阶段未采用release，那么字节码怎样链接都不会变，只有重定位。  
&emsp;&emsp;经我实验，发现lib文件中的模块在链接时会发生重定位，因此call ??? jmp ???字节码相应地会发生调整，于是IDA就无法处理了。我想如果在这方面有所修改，识别能力将大大加强，不过会损失一些速度和产生误判。  
&emsp;&emsp;strcpy memcpy等内联函数和库函数，这种是不存在静态库中而是内联的话，稍经优化IDA也会无法识别，导致大量函数无法识别。  
&emsp;&emsp;不要太把FLIRT技术太当回事，等模糊匹配技术发展起来吧！  
&emsp;&emsp;如果有好的意见和建议不妨提出来，以上仅为我个人观点
