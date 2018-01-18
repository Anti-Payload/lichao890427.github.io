---
layout: post
title: OllyDbg插件深入分析
categories: Reverse_Engineering
description: OllyDbg插件深入分析
keywords: ollydbg
---

# OllyDbg插件深入分析

## 第一章 初探OllyDbg插件

&emsp;&emsp;官网<http://www.ollydbg.de/>给出了关于插件开发的信息。OllyDbg1.10的插件开发包在<http://www.ollydbg.de/plug11.zip>。该压缩包包含以下文件：

|File       |Description                                         |
|-----------|----------------------------------------------------|
|Bookmark.c |OllyDbg书签插件源码，该插件支持调试程序时设置10个书签   |
|Cmdexec.c  |OllyDbg命令行插件，该插件支持输入命令进行调试          |
|Cmdline.rtf| 命令行插件的帮助文件                                 |
|Command.c  |                                                    |
|Ollydbg.def|OllyDbg定义文件，某些编译器用之生成输入链接库ollydbg.li|
|Plugin.h   |插件公共头文件                                       |
|Plugins.hlp|插件编写说明                                         |

&emsp;&emsp;插件开发目录结构如下：
```Txt
│
├─Bc55                   Borland C++系列编译器工程
│      BOOKMARK.MAK
│      CMDLINE.BPR
│      CMDLINE.CPP
│      CMDLINE.MAK
│      OLLYDBG.LIB
│      SAMPLE.BPR
│      SAMPLE.CPP
│
└─Vc50                  Visual C++系列编译器工程，这也是本文所使用的开发环境
        BOOKMARK.DSP
        BOOKMARK.DSW
        BOOKMARK.MAK
        CMDLINE.DSP
        CMDLINE.DSW
        CMDLINE.MAK
        OLLYDBG.LIB       
```

### 一、基本原理

&emsp;&emsp;OllyDbgv1.10是OllyDbg1系列的最终版本，作者已停止开发，转而开发v2.0版本，新版本和1.xx版本是不兼容的，插件也是如此。对于1.xx版本，插件大体上通用，这几个版本的改动有：  
* t_reg和t_bpoint结构体扩展  
* 新选项“总在最前”需要插件窗口特殊支持  
* Browsefilename支持保存文件对话框  
&emsp;&emsp;插件是提供附加功能的DLL文件，位于OllyDbg目录下。OllyDbg启动时会逐个加载所有可用的DLL文件，检查名为_ODBG_Plugindata和_ODBG_Plugininit的入口点（输出函数），如果存在并且插件版本号兼容，OllyDbg会注册插件并在插件子菜单增加相应项。插件可以在反汇编、转储、堆栈、内存、模块、线程、断点、监视、参考、界面窗口、运行跟踪窗口增加菜单项和监视全局/局部快捷键。插件可以是MDI窗口；可以在.udd文件中写入模块相关的自定义数据；可以访问和修改ollydbg.ini的数据结构以描述调试信息。插件使用多个回调函数和OllyDbg通信，可以调用170+个插件API函数。插件接口不是面向对象的。插件API函数不是线程安全的，没有实现临界区，插件创建的新线程不能调用这些函数，否则可能导致OllyDbg和程序崩溃。  

### 二、编译

&emsp;&emsp;请将编译器按如下设置以便插件和OllyDbg通信，plugin.h会检查这些设置：
* 如果使用C++编译器则需要禁用导出函数的名称修饰（使用extern “C”）  
* 强制所有API函数和导出函数使用C格式调用_cdecl  
* 强制所有结构体按字节对齐  
* 默认字符类型为unsigned型  

&emsp;&emsp;编译自定义插件会用到plugin.h ollydbg.lib，需要复制到工程目录中。现在以VS2010为例介绍如何编写无任何功能的插件。首先建立一个Windows动态链接库的空项目，在工程属性中，选C/C++ -> 命令行，右侧“其它选项”加入“/J”。之后向工程添加helloworld.cpp，内容如下：

```C++
#include <windows.h>
#include "lugin.h"
HINSTANCE hinst=NULL;
BOOL WINAPIDllEntryPoint(HINSTANCE hi,DWORD reason,LPVOID reserved)
{//DLL入口点
       if (reason==DLL_PROCESS_ATTACH)
                hinst=hi; 
       return 1; 
}
extc int _export cdecl ODBG_Plugininit(int ollydbgversion,HWND hw,ulong *features)
{
       MessageBox(NULL,"HelloWorld","HelloWorld",MB_OK);
       return 0;
}
extc int _export cdecl ODBG_Plugindata(char shortname[32]) 
{//用于插件菜单中显示插件名
       strcpy(shortname,"HelloWorld");   
       return PLUGIN_VERSION;
}
```

&emsp;&emsp;编译得到的dll放到ollydbg根目录下，启动ollydbg就可以看到弹出对话框，同时插件菜单栏多了一项“Hello Wolrd”。使用dumpbin或depends工具查看ollydbg.exe输出表，可以看到700+个函数，这些就是插件API函数。ollydbg.exe导出了很多函数目的是提供插件插件，将如设置断点、反编译等一些相对独立的模块，抽取出来，可供第三方调用。调用OllyDbg-API：欲调用OllyDbg导出函数（例如FuncA），首先在源文件中包含Plugin.h，并在调用之前增加代码“#pragmacomment(lib,"ollydbg.lib")”，同时将插件开发包Vc50目录下的ollydbg.lib拷贝到工程目录中。此外OllyDbg作者写的Plugin.h不适用于VS系列编译器，由于ollydbg.exe实际导出符号为下划线版本（_FuncA），而plugin.h声明的是无下划线形式，因此直接编译会出现链接问题，而作者仅对ODBG系列函数作出调整而未对OllyDbg-API声明作出相应调整，因此所有用到的OllyDbg-API都需要进行手工调整，先找到Plugin.h中这样的代码段：

```C++
  #define ODBG_Plugindata      _ODBG_Plugindata
  #define ODBG_Plugininit      _ODBG_Plugininit
  #define ODBG_Pluginmainloop  _ODBG_Pluginmainloop
  #define ODBG_Pluginsaveudd   _ODBG_Pluginsaveudd
  #define ODBG_Pluginuddrecord_ODBG_Pluginuddrecord
  #define ODBG_Pluginmenu      _ODBG_Pluginmenu
  #define ODBG_Pluginaction    _ODBG_Pluginaction
  #define ODBG_Pluginshortcut  _ODBG_Pluginshortcut
  #define ODBG_Pluginreset     _ODBG_Pluginreset
  #define ODBG_Pluginclose     _ODBG_Pluginclose
  #define ODBG_Plugindestroy   _ODBG_Plugindestroy
  #define ODBG_Paused          _ODBG_Paused
  #define ODBG_Pausedex        _ODBG_Pausedex
  #define ODBG_Plugincmd       _ODBG_Plugincmd
```

#### 头文件包含接口声明
&emsp;&emsp;在其后加入自己的声明这样方可正常编译链接，如：

```C++
  #define Plugingetvalue                         _Plugingetvalue
  #define Getstatus                              _Getstatus
```

#### 获取导入库

&emsp;&emsp;之后手动生成ollydbg.lib：ollydbg.lib可以由插件根目录存在ollydbg.def文件手动生成，这里会用到VS编译器自带工具lib.exe，命令如下：`lib/MACHINE:X86 /DEFllydbg.def`

#### 编译调试

&emsp;&emsp;调试：写插件本身具有难度，然而调试OllyDbg运行插件似乎就更难了。然而我却不以为然，将OllyDbg拷贝到生成dll的目录中（前提是该版本OllyDbg读取插件的目录为自身根目录），设置工程属性=>调试=>命令，将拷贝后的OllyDbg文件路径写入该处，调试即可在DLL源码中断下。

### 三、使用MFC开发OllyDbg1插件

&emsp;&emsp;上面介绍的是使用MSVC的Windows DLL工程的情况，而这里介绍如何结合MFC进行插件开发。经我测试，VS2010及之后的MFC，由于内部使用的ATL和/J编译指令冲突，因此无法编译，而VC6版本可以很好地编译。下面是开发步骤，以test为例：
* 1.新建名为test的MFC DLL工程
* 2.在自动生成的StdAfx.h文件末尾加入 #inclue “Plugin.h” 同时将Plugin.h拷入工程目录
* 3.在test.cpp中添加ODBG_***导出函数
* 4.在调用OllyDbg导出函数之前，加入#pragma comment(lib,"ollydbg.lib")

### 四、插件生命周期
&emsp;&emsp;OllyDbg所规定的插件输出函数，其实正好反映了插件的生命周期，类似于窗口的生命周期，它与消息机制相关。下面通过实例得到插件生命周期：

```C++
#include <windows.h>
#include "lugin.h"
extc int _export cdecl ODBG_Plugindata(char shortname[32]) 
{//插件检测
       strcpy(shortname,"菜单显示项");  
       MessageBox(NULL,"ODBG_Plugindata","",MB_OK);
       return PLUGIN_VERSION;
}
extc int _export cdecl ODBG_Plugininit(int ollydbgversion,HWND hw,ulong *features)
{//插件初始化
       MessageBox(NULL,"ODBG_Plugininit","",MB_OK);
       return 0;
}
extc int  _export cdecl ODBG_Pluginmenu(int origin,char data[4096],void *item)
{//初始化菜单项
       MessageBox(NULL,"ODBG_Pluginmenu","",MB_OK);
       return 0;
}
extc int  _export cdecl ODBG_Pluginclose(void)
{//用户关闭OllyDbg时触发
       MessageBox(NULL,"ODBG_Pluginclose","",MB_OK);
       return 0;
}
extc void _export cdecl ODBG_Plugindestroy(void)
{//OllyDbg退出时触发
       MessageBox(NULL,"ODBG_Plugindestroy","",MB_OK);
}
// 输出 ODBG_Plugindata=> ODBG_Plugininit => ODBG_Pluginmenu => ODBG_Pluginclose => ODBG_Plugindestroy
```

## 第二章 OllyDbg输出函数按功能分类

### 熟悉插件函数

&emsp;&emsp;学习Windows编程需要熟悉WindowsAPI，同样地，熟悉OllyDbg插件编程也要熟悉OllyDbg API。

原型：`int Registerpluginclass(char *classname,char *iconname,HINSTANCE dllinst,WNDPROCclassproc)`   
功能：生成唯一的类名并注册为插件窗口。如果iconname为NULL则使用标准插件图标（字母P）  
返回：成功时返回0并填充classname，失败时返回1  
参数：  
&emsp;&emsp;classname 指向大小大于32字符的缓冲区用于接收类名  
&emsp;&emsp;iconname 插件DLL中图标资源名  
&emsp;&emsp;dllinst 插件实例句柄  
&emsp;&emsp;classproc 新类的窗口过程地址  
注意：注册后窗口类有8个整数（32字节）的额外空间，插件可以自由使用第2到7个整数空间（偏移8到28用于GetWindowLong和SetWindowLong）。ODBG_Plugininit是执行该函数的最佳位置  

原型：`void Unregisterpluginclass(char *classname)`  
功能：注销之前通过Registerpluginclass注册的窗口类  
参数：  
&emsp;&emsp;classname 函数Registerpluginclass返回的类名  
注意：在ODBG_Plugindestroy中对所有注册的窗口类调用该函数  

原型：`int Pluginwriteinttoini(HINSTANCE dllinst,char *key,int value)`  
功能：将整数值绑定的键值对存储在ollydbg.ini的插件自定义区段中  
返回：成功时返回1，失败时返回0  
参数：  
&emsp;&emsp;dllinst 插件实例句柄  
&emsp;&emsp;key 整数相关的键名  
&emsp;&emsp;value 要存储在ollydbg.ini的整数  

原型：`int Pluginwritestringtoini(HINSTANCE dllinst,char *key,char *s)`  
功能：将ASCII字符串绑定的键值对存储在ollydbg.ini的插件自定义区段中  
返回：成功时返回1，失败时返回0  
参数：  
&emsp;&emsp;dllinst 插件实例句柄  
&emsp;&emsp;key 字符串相关的键名  
&emsp;&emsp;s 要存储在ollydbg.ini的字符串  

原型：`int Pluginreadintfromini(HINSTANCE dllinst,char *key,int def)`  
功能：将整数值绑定的键值对从ollydbg.ini的插件自定义区段中读取出来  
返回：成功时返回目标整数，失败时返回默认值  
参数：  
&emsp;&emsp;dllinst 插件实例句柄  
&emsp;&emsp;key 整数相关的键名  
&emsp;&emsp;def 默认值  

原型：`int Pluginreadstringfromini(HINSTANCE dllinst,char *key,char *s,char *def)`  
功能：将字符串绑定的键值对从ollydbg.ini的插件自定义区段中读取出来  
返回：成功时返回目标字符串，失败时返回默认值  
参数：  
&emsp;&emsp;dllinst 插件实例句柄  
&emsp;&emsp;key 字符串相关的键名  
&emsp;&emsp;s 用于接收目标字符串  
&emsp;&emsp;def 默认字符串，以零终止符结尾  

原型：`int Pluginsaverecord(ulong tag,ulong size,void *data)`  
功能：将一个记录写入.udd文件  
返回：成功时返回1，失败时返回0  
参数：  
&emsp;&emsp;tag 插件唯一的标签  
&emsp;&emsp;size 写入.udd文件的数据大小，最大USERLEN  
&emsp;&emsp;data 写入.udd文件的数据缓冲区  
注意：只能从ODBG_Pluginsaveudd中调用，否则会崩溃  

原型：`int Plugingetvalue(int type)`  
功能：返回OllyDbg多个设置和变量信息  
参数：  
&emsp;&emsp;type 要返回的设置或变量信息  

|type           |实际类型    |含义                |
|---------------|-----------|--------------------|
|VAL_HINST      |HINST      |当前OllDbg实例句柄   |
|VAL_HWMAIN     |HWND       |OllyDbg主窗口句柄    |
|VAL_HWCLIENT   |HWND       |MDI用户窗口句柄      |
|VAL_NCOLORS    |int        |常见颜色数           |
|VAL_COLORS     |COLORREF*  |常见颜色RGB值数组    |
|VAL_BRUSHES    |HBRUSH*    |常见颜色画刷句柄数组  |
|VAL_PENS       |HPEN*      |常见颜色画笔句柄数组  |
|VAL_NFONTS     |int        |常见字体数           |
|VAL_FONTS      |HFONT*     |常见字体句柄数组      |
|VAL_FONTNAMES  |Char**     |内部字体名称         |
|VAL_FONTWIDTHS |int*       |常见字体平均宽度      |
|VAL_FONTHEIGHTS|int*       |常见字体平均高度      |
|VAL_NFIXFONTS  |int        |固定字宽字体数        |
|VAL_DEFFONT    |int        |默认字体序号         |
|VAL_NSCHEMES   |int        |配色方案数            |
|VAL_SCHEMES    |t_scheme*  |配色方案数组         |
|VAL_DEFSCHEME  |           |默认配色方案数组      |
|VAL_DEFHSCROLL |           |默认水平滚动         |
|VAL_RESTOREWINDOWPOS|      |从.ini文件恢复窗口位置|
|VAL_HPROCESS   |HANDLE     |被调试进程句柄       |
|VAL_PROCESSID  |           |被调试进程标志ID     |
|VAL_HMAINTHREAD|HANDLE     |被调试进程主线程句柄 |
|VAL_MAINTHREADID|          |被调试进程主线程标志ID|
|VAL_MAINBASE   |           |被调试进程主模块基址 |
|VAL_PROCESSNAME|char*      |被调试进程名         |
|VAL_EXEFILENAME|char*      |被调试程序文件名     |
|VAL_CURRENTDIR |char*      |被调试进程当前目录   |
|VAL_SYSTEDIR   |char*      |系统目录            |
|VAL_DECODEANYIP|           |不依赖EIP解码寄存器  |
|VAL_PASCALSTRINGS|         |解码Pascal格式字符串 |
|VAL_ONLYASCII  |           |只解码可打印ASCII字符|
|VAL_DIACRITICALS|          |允许字符串有变音符号 |
|VAL_GLOBALSEARCH|          |从块的开始处搜索     |
|VAL_ALIGNEDSEARCH|         |逐项搜索            |
|VAL_SEARCHMARGIN|          |浮点搜索允许误差     |
|VAL_KEEPSELSIZE|           |保存16进制编辑中选择项数|
|VAL_MMXDISPLAY|            |对话框MMX显示模式(0:16进制 1:有符号 2:无符号)|
|VAL_WINDOWFONT|            |对话框中使用窗口字体 |
|VAL_TABSTOPS               |制表符大小          |
|VAL_MODULES   |t_table*    |模块表(包括.EXE和.DLL)|
|VAL_MEMORY    |t_table*    |分配内存块表         |
|VAL_THREADS   |t_table*    |活动线程表           |
|VAL_BREAKPOINTS|t_table*   |激活断点表           |
|VAL_REFERENCES|t_table*    |查找到的参考信息表    |
|VAL_SOURCELIST|t_table*    |源文件表             |
|VAL_WATCHES    |t_table*   |监视情况表           |
|VAL_CPUFEATURES|           |CPUID返回的CPU特征位 |
|VAL_TRACEFILE  |FILE*      |运行跟踪记录文件句柄  |
|VAL_ALIGNDIALOGS|          |对齐对话框           |
|VAL_CPUDASM    |t_dump*    |获取CPU反汇编窗口描述符|
|VAL_CPUDDUMP   |t_dump*    |获取CPU内存窗口描述符 |
|VAL_CPUDSTACK  |t_dump*    |获取CPU栈窗口描述符   |
|VAL_APIHELP    |char*      |选择的API帮助文件名   |
|VAL_HARDBP     |           |硬件断点是否激活      |
|VAL_PATCHES    |t_table*   |补丁表               |
|VAL_HINTS      |t_sorted*  |带分析提示的排序数据   |
     
&emsp;&emsp;说明：带有VAL_N的type变量表示获取数量，需要先获取该数量才能知道对应的VAL数据数组有多大。例如FONT系列，首先要调用Plugingetvalue(VAL_NFONTS)获取字体个数，之后再调用VAL_FONTNAMES等获取数据数组，数组元素个数就是前一步获取的个数。t_table和t_dump也类似，t_table内含的重要数据是t_sorteddata，而t_sorted结构体自身包含了元素个数，因此读者可根据例1写出输出更完整信息的程序了。返回t_dump类型的函数大部分是窗口数据，从这个意义上插件可以做的和OllyDbg一模一样！

原型：`t_status Getstatus(void)`  
功能：返回被调试进程的当前状态（STAT_XXX）  
返回：  
&emsp;&emsp;STAT_NONE 未调试进程  
&emsp;&emsp;STAT_STOPPED 进程挂起  
&emsp;&emsp;STAT_EVENT 处理调试事件，进程暂停  
&emsp;&emsp;STAT_RUNNING 进程运行  
&emsp;&emsp;STAT_FINISHED 进程结束  
&emsp;&emsp;STAT_CLOSING 调用TerminateProcess()等待结果
   
### 例子

```C++
#include <windows.h>
#include <stdio.h>
#include "Plugin.h"
#pragma comment(lib,"OllyDbg.lib")
char classname[32];
HINSTANCE hinst=NULL;
class log//用于实现插件日志记录——单例模式
{
private:
        log()
        {
                 try
                 {
                         if(!AllocConsole())
                         {
                                  MessageBox(NULL,"无法创建命令行窗口","错误",MB_OK);
                                  throw "错误";
                         }
                 }
                 catch(...)
                 {
                         exit(0);
                 }
        };
        ~log()
        {
                 FreeConsole();
        }
public:
        static void v(char* tolog)
        {
                 static log obj;
                 HANDLEout=GetStdHandle(STD_OUTPUT_HANDLE);
                 DWORDWriteNum;
                 WriteConsole(out,tolog,strlen(tolog),&WriteNum,NULL);
        }
};
BOOL WINAPI DllMain(HINSTANCE hi,DWORD reason,LPVOID reserved)
{//DLL入口点
        if (reason==DLL_PROCESS_ATTACH)
        {
                 hinst=hi;//保存实例句柄供后面使用
        }
        return 1; 
}
LRESULT CALLBACK WndProc(HWND hw,UINT msg,WPARAM wp,LPARAM lp)
{
        return DefWindowProc(hw,msg,wp,lp);//暂时不用这里，因此只写个空架子
}
extc int _export cdecl ODBG_Plugininit(int ollydbgversion,HWND hw,ulong *features)
{
        chartemp[256];
        sprintf(temp,"版本号：%d\n",ollydbgversion);
        log::v(temp);
        if(0 == Registerpluginclass(temp,NULL,hinst,WndProc))
        {
                 log::v("插件注册成功：");
                 log::v(temp);
                 log::v("\n");
        }
        else
        {
                 log::v("插件注册失败\n");
        }
        return 0;
}
extc int _export cdecl ODBG_Plugindata(char shortname[32]) 
{//用于插件菜单中显示插件名
        strcpy(shortname,"sample1");   
        return PLUGIN_VERSION;
}
extc int _export cdecl ODBG_Pluginshortcut(int origin,int ctrl,int alt,int shift,int key,void *item)
{//按下I键（代表Information）显示OLLYDBG和进程信息
                 if (key!='I')
                         return 0;
                 switch(Getstatus())
                 {
                         case STAT_NONE:
                                  log::v("——————————————未调试进程——————————————\n");
                                  break;
                         case STAT_STOPPED:
                                  log::v("——————————————进程挂起——————————————\n");
                                  break;
                         case STAT_EVENT:
                                  log::v("——————————————进程暂停，处理调试事件——————————————\n");
                                  break;
                         case STAT_RUNNING:
                                  log::v("——————————————进程运行——————————————\n");
                                  break;
                         case STAT_FINISHED:
                                  log::v("——————————————进程结束——————————————\n");
                                  break;
                         case STAT_CLOSING:
                                  log::v("——————————————进程等待关闭——————————————\n");
                                  break;
                 }
                 chartemp[256];
                 sprintf(temp,"OllyDbg实例句柄：0x%0x\n",Plugingetvalue(VAL_HINST));
                 log::v(temp);
                 sprintf(temp,"OllyDbg主窗口句柄：0x%0x\n",Plugingetvalue(VAL_HWMAIN));
                 log::v(temp);
                 sprintf(temp,"被调试进程句柄：0x%0x\n",Plugingetvalue(VAL_HPROCESS));
                 log::v(temp);
                 sprintf(temp,"被调试进程ID：%d\n",Plugingetvalue(VAL_PROCESSID));
                 log::v(temp);
                 sprintf(temp,"被调试进程主线程句柄：0x%0x\n",Plugingetvalue(VAL_HMAINTHREAD));
                 log::v(temp);
                 sprintf(temp,"被调试进程主线程ID：%d\n",Plugingetvalue(VAL_MAINTHREADID));
                 log::v(temp);
                 sprintf(temp,"被调试进程主模块基址：0x%0x\n",Plugingetvalue(VAL_MAINBASE));
                 log::v(temp);
                 sprintf(temp,"被调试进程名：%s\n",Plugingetvalue(VAL_PROCESSNAME));
                 log::v(temp);
                 sprintf(temp,"被调试进程文件名：%s\n",Plugingetvalue(VAL_EXEFILENAME));
                 log::v(temp);
                 sprintf(temp,"被调试进程当前目录：0x%0x\n",Plugingetvalue(VAL_CURRENTDIR));
                 log::v(temp);
                 sprintf(temp,"系统目录：%s\n",Plugingetvalue(VAL_SYSTEMDIR));
                 log::v(temp);
                 sprintf(temp,"被调试进程主模块基址：0x%0x\n",Plugingetvalue(VAL_MAINBASE));
                 log::v(temp);
};
```

## 第三章 对于OllyDbg加载插件过程的分析

### 初探

&emsp;&emsp;这里对v1.10版本进行逆向，加载插件是在OllyDbg启动后，加载调试程序前完成的，因此需要用OllyDbg调试自身。文档中说OllyDbg先检查_ODBG_Plugindata，那么下断点bp GetProcAddress,[esp+8]=="_ODBG_Plugindata"会发现断在以地址0x00496658开始的函数中。查看其反汇编代码，先进行总体分析，发现有多处调用GetProcAddress，初步判断为加载插件的模块，命名为LoadPlugin，经过分析代码如下：

```C++
#include <windows.h>  
#include <dos.h>  
#include "plugin.h"  
struct PluginData  
{  
    HMODULE hPluginDll;  
    char    DllName[260];  
    char PluginName[32];  
//+296  
    ???  
//+560  
    ODBG_Pluginmainloop;  
    ODBG_Pluginmenu;  
    ODBG_Pluginaction;  
    ODBG_Pluginshortcut;  
    ODBG_Pluginsaveudd;  
    ODBG_Pluginuddrecord;  
    ODBG_Pluginreset;  
    ODBG_Paused;  
    ODBG_Pausedex;  
    ODBG_Plugincmd;  
};  
   
int pluginnum;  
PluginData plugindata[32];//最多32个插件  
char data[0x1100];  
HANDLE hwmain;  
   
bool LoadPlugins()  
{  
    char pluginpath[260],filename[256],pluginname[32];  
    HANDLE hFindFile;  
    WIN32_FIND_DATA FindFileData;  
    HMENU pluginmenu,popupmenu;  
    HMODULE hmod;  
    int ret;  
    int pluginmenuid;  
   
    memset(plugindata,sizeof(plugindata));  
    pluginnum=0;  
    strcpy(pluginpath,"*.dll");  
    hFindFile=FindFirstFile(pluginpath,&FindFileData);  
    if(hFindFile == INVALID_HANDLE_VALUE)  
        return false;  
    pluginmenu=CreateMenu();  
    if(!pluginmenu)  
        return false;  
    do  
    {//搜索根目录下所有dll文件  
        hmod=NULL;  
        fnsplit(FindFileData.cFileName,NULL,NULL,filename,NULL);  
        if(stricmp(filename,"psapi") && stricmp(filename,"dbghelp"))  
        {//如果不是psapi.dll和dbghelp.dll  
            strcpy(pluginpath,FindFileData.cFileName);  
            hmod=LoadLibrary(pluginpath);  
            if(hmod)  
            {  
                ODBG_Plugindata=GetProcAddress(hmod,"_ODBG_Plugindata");  
                ODBG_Plugininit=GetProcAddress(hmod,"_ODBG_Plugininit");  
                if(ODBG_Plugindata && ODBG_Plugininit)  
                {  
                    pluginname[0]='\0';  
                    ret=ODBG_Plugindata(pluginname);  
                    if(ret >= 106 && ret <= 110 && pluginname[0] != '\0')//版本在1.06~1.10之间  
                    {  
                        PluginData& curplugin=plugindata[pluginnum];  
                        curplugin.hPluginDll=hmod;  
                        strcpy(curplugin.DllName,FindFileData.cFileName);  
                        strncpy(curplugin.PluginName,pluginname,31);  
                        curplugin.PluginName[31]='\0';  
                        curplugin.ODBG_Pluginaction=GetProcAddress(hmod,"ODBG_Pluginaction");  
                        curplugin.ODBG_Pluginmainloop=GetProcAddress(hmod,"ODBG_Pluginmainloop");  
                        curplugin.ODBG_Pluginmenu=GetProcAddress(hmod,"ODBG_Pluginmenu");  
                        curplugin.ODBG_Pluginshortcut=GetProcAddress(hmod,"ODBG_Pluginshortcut");  
                        curplugin.ODBG_Pluginsaveudd=GetProcAddress(hmod,"ODBG_Pluginsaveudd");  
                        curplugin.ODBG_Pluginuddrecord=GetProcAddress(hmod,"ODBG_Pluginuddrecord");  
                        curplugin.ODBG_Pluginreset=GetProcAddress(hmod,"ODBG_Pluginreset");  
                        curplugin.ODBG_Paused=GetProcAddress(hmod,"ODBG_Paused");  
                        curplugin.ODBG_Pausedex=GetProcAddress(hmod,"ODBG_Pausedex");  
                        curplugin.ODBG_Plugincmd=GetProcAddress(hmod,"ODBG_Plugincmd");  
                        ulong feature=0;  
                        ret=ODBG_Plugininit(110,hwmain,&feature);  
                        if(ret)  
                        {  
                            Addtolist(0,0,"Plugin '%s' failed to initialize (code %i)",filename,ret);  
                        }  
                        else 
                        {  
                            pluginmenuid=pluginnum*32+57344;  
                            pluginname[0]='\0';  
                            if(curplugin.ODBG_Pluginmenu) && curplugin.ODBG_Pluginmenu(PM_MAIN,data,NULL))  
                            {  
                                if(pluginname[0] != '\0' && (popupmenu=CreateMenu()) != NULL)  
                                {  
                                    CreateSubMenu(popupmenu,curplugin,pluginmenuid,1);  
                                }  
                                if(pluginnum >= 10)  
                                    sprintf(pluginname,"%s",curplugin.pluginname);  
                                else 
                                    sprintf(pluginname,"&%i %s",(pluginnum+1)%10,curplugin.pluginname);  
                                if(popupmenu)  
                                    AppendMenu(pluginmenuid,MF_POPUP,popupmenu,pluginname);  
                                else 
                                    AppendMenu(pluginmenuid,0,pluginmenuid,pluginname);  
                                pluginnum++;  
                                hmod=NULL;  
                            }  
                        }  
                    }  
                    else 
                    {  
                        Addtolist(0,0,"Plugin '%s' has invalid version (%i.%02i)",filename,ret/100,ret%100);  
                    }  
                }  
            }  
        }  
        if(hmod)  
            FreeLibrary(hmod);  
    }   
    while(FindNextFile(hFindFile,&FindFileData));  
} 
```

可见加载过程是：ODBG_Plugindata => ODBG_Plugininit => ODBG_Pluginmenu

### 深入

&emsp;&emsp;看完了API级别基本用法，现在来看对应OllyDbg内部实现吧！这样可以和第二章作对照这里只进行简单分析即贴出C语言源码，如对逆向分析感兴趣，请关注我的书。以下部分每一节对应第二章的各个节，实为深入底层研究。加载插件是在OllyDbg启动后，加载调试程序前完成的，因此需要用OllyDbg调试自身。文档中说OllyDbg先检查_ODBG_Plugindata，那么下断点`bp Kernel32.GetProcAddress,[esp+8]=="_ODBG_Plugindata"`会发现断在以地址0x00496658开始的函数中。查看其反汇编代码，先进行总体分析，发现有多处调用GetProcAddress，初步判断为加载插件的模块，命名为LoadPlugin，逆向分析代码如下：

#### LoadPlugin

```C++
int pluginnum;
PluginData plugindata[32];//最多32个插件
char data[0x1100];
HANDLE hwmain;
bool LoadPlugins()
{
	charpluginpath[260],filename[256],pluginname[32];
	HANDLEhFindFile;
	WIN32_FIND_DATAFindFileData;
	HMENUpluginmenu,popupmenu;
	HMODULE hmod;
	int ret;
	intpluginmenuid;
	memset(plugindata,sizeof(plugindata));
	pluginnum=0;
	strcpy(pluginpath,"*.dll");
	hFindFile=FindFirstFile(pluginpath,&FindFileData);
	if(hFindFile== INVALID_HANDLE_VALUE)
			return false;
	pluginmenu=CreateMenu();
	if(!pluginmenu)
			return false;
	do 
	{//搜索根目录下所有dll文件
		hmod=NULL;
		fnsplit(FindFileData.cFileName,NULL,NULL,filename,NULL);
		if(stricmp(filename,"psapi")&& stricmp(filename,"dbghelp"))
		{//如果不是psapi.dll和dbghelp.dll
			strcpy(pluginpath,FindFileData.cFileName);
			hmod=LoadLibrary(pluginpath);
			if(hmod)
			{
				ODBG_Plugindata=GetProcAddress(hmod,"_ODBG_Plugindata");
				ODBG_Plugininit=GetProcAddress(hmod,"_ODBG_Plugininit");
				if(ODBG_Plugindata&& ODBG_Plugininit)
				{
					pluginname[0]='\0';
					ret=ODBG_Plugindata(pluginname);
					if(ret>= 106 && ret <= 110 && pluginname[0] != '\0')//版本在1.06~1.10之间
					{
						PluginData&curplugin=plugindata[pluginnum];
						curplugin.hPluginDll=hmod;
						strcpy(curplugin.DllName,FindFileData.cFileName);
						strncpy(curplugin.PluginName,pluginname,31);
						curplugin.PluginName[31]='\0';
						curplugin.ODBG_Pluginaction=GetProcAddress(hmod,"ODBG_Pluginaction");
						curplugin.ODBG_Pluginmainloop=GetProcAddress(hmod,"ODBG_Pluginmainloop");
						curplugin.ODBG_Pluginmenu=GetProcAddress(hmod,"ODBG_Pluginmenu");
						curplugin.ODBG_Pluginshortcut=GetProcAddress(hmod,"ODBG_Pluginshortcut");
						curplugin.ODBG_Pluginsaveudd=GetProcAddress(hmod,"ODBG_Pluginsaveudd");
						curplugin.ODBG_Pluginuddrecord=GetProcAddress(hmod,"ODBG_Pluginuddrecord");
						curplugin.ODBG_Pluginreset=GetProcAddress(hmod,"ODBG_Pluginreset");
						curplugin.ODBG_Paused=GetProcAddress(hmod,"ODBG_Paused");
						curplugin.ODBG_Pausedex=GetProcAddress(hmod,"ODBG_Pausedex");
						curplugin.ODBG_Plugincmd=GetProcAddress(hmod,"ODBG_Plugincmd");
						ulongfeature=0;
						ret=ODBG_Plugininit(110,hwmain,&feature);
						if(ret)
						{
							Addtolist(0,0,"Plugin'%s' failed to initialize (code %i)",filename,ret);
						}
						else
						{
							pluginmenuid=pluginnum*64+57344;
							pluginname[0]='\0';
							if(curplugin.ODBG_Pluginmenu)&& curplugin.ODBG_Pluginmenu(PM_MAIN,data,NULL))
							{
								if(pluginname[0]!= '\0'&& (popupmenu=CreateMenu()) != NULL)
								{
									CreateSubMenu(popupmenu,curplugin,pluginmenuid,1);
								}
								if(pluginnum>= 10)
									sprintf(pluginname,"%s",curplugin.pluginname);
								else
									sprintf(pluginname,"&%i%s",(pluginnum+1)%10,curplugin.pluginname);
								if(popupmenu)
									AppendMenu(pluginmenuid,MF_POPUP,popupmenu,pluginname);
								else
									AppendMenu(pluginmenuid,0,pluginmenuid,pluginname);
								pluginnum++;
								hmod=NULL;
							}
						}
					}
					else
					{
						Addtolist(0,0,"Plugin'%s' has invalid version (%i.%02i)",filename,ret/100,ret%100);
					}
				}
			}
		}
		if(hmod)
			FreeLibrary(hmod);
	} 
    while(FindNextFile(hFindFile,&FindFileData));
}
```

&emsp;&emsp;可见加载过程是：ODBG_Plugindata => ODBG_Plugininit => ODBG_Pluginmenu，同理可分析其它函数。

#### ODBG_Pluginmainloop

&emsp;&emsp;继续来看ODBG_Pluginmainloop函数，如何断在插件中该函数入口呢，在这里我利用GetProcAddress返回值，先`bp Kernel32.GetProcAddress,[esp+8]=="_ODBG_Pluginmainloop"`，Ctrl+F9执行到返回，再单步一次即可跳出GetProcAddress函数到ollydbg函数中，此时eax为返回值为获取到的函数地址，因此bp eax可以断在ODBG_Pluginmainloop函数内，运行后程序果然断在其中：

```C++
voidCallEverymainloop(DEBUG_EVENT *debugevent)
{
       for(int i=0;i<pluginnum;i++)
       {
              if(plugindata.ODBG_Pluginmainloop)
              plugindata. ODBG_Pluginmainloop(debugevent);
       }
}
```

&emsp;&emsp;继续跳出后发现即是主函数WinMain且处于消息循环代码中（如果使用IDA查看调用关系，会发现Suspendprocess和Injectcode函数中均调用了该函数。这里不做详解），分析后得到：

```C++
if(procstatus != STAT_RUNNING)
{
       CallEverymainloop(NULL);
       Sleep(1);
}
```

#### ODBG_pluginaction

```C++
bool CallEveryaction(int origin,int resourceid,void*item)
{
        intpluginindex;
        if(resourceid< 57344)
                 returnfalse;
        pluginindex=(resourceid-57344)/64;//由菜单资源id得到插件序号，和前面插件加载过程相对应
        if(pluginindex>= pluginnum || plugindata[pluginindex].ODBG_pluginaction== NULL)
                 returnfalse;
        plugindata[pluginindex].ODBG_pluginaction(origin,resourceid-pluginindex*64+57344,item);
       returntrue;
}
```

#### ODBG_Pluginshortcut

```C++
LRESULT CALLBACKWndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
        ......
        switch(message)
        {
                 ……
                 caseWM_KEYDOWN:
                 caseWM_SYSKEYDOWN:
                         return CallEveryshortcut(PM_MAIN,GetKeyState(VK_CONTROL)&0x8000,
message==WM_SYSKEYDOWN,GetKeyState(VK_SHIFT)&0x8000,wParam,NULL);
                         break;
                 ……
        }
        ......
}

int CallEveryshortcut(int orgin,bool ctrl,bool alt,boolshift,int key,void* item)
{
        if(key ==VK_SHIFT || key == VK_CONTROL || key == VK_MENU)//单个键无效
                 return0;
        for(intpluginindex=0;pluginindex<pluginnum;pluginindex++)
        {
         if(plugindata[pluginindex].ODBG_Pluginshortcut(origin,ctrl,alt,shift,key,item))
                     return1;
       }
       return0;
}

```

## 第四章 OllyDbg插件CheatUtility逆向分析

&emsp;&emsp;该插件可以自由修改程序数据且在数据处下断点以捕获修改代码，活像OllyDbg的CheatEngine。含2个对话框，插件目录下可以看到作者特意做了个使用视频，确实是很有用的工具。文件大小11kb，用户代码部分3828b，译得500行c代码，关键反汇编代码如下：

### 逆向

```C++
#define IDC_BTFIRST 100//First Scan Button
#define IDC_BTNEXT 101//Next Scan Button
#define IDC_BTSTOP 102//Stop Button
#define IDC_ETVAL 200//Value Edit
#define IDM_CHANGE 500//Change Value
#define IDM_FOLLOW 501//Follow in Dump
#define IDM_HARDWARE 502//Hardware Breakpoint
#define IDM_DELETE 503//Delete Button
#define IDD_MAINDLG 1000//Main Dialog
#define IDC_LVLIST 1001//Address ListView
#define IDC_SBSTATU 1004//Bottom Statu Bar
#define IDC_CBISHEX 1011//Is Hex Check Button
#define IDC_RBDWORD 1014//DOUBLE WORD Radio
#define IDC_RBWORD 1015//WORD Radio
#define IDC_RBBYTE 1016//BYTE Radio
#define IDD_SETDIALOG 1027//Set Dialog
#define IDC_BTOK 1034//OK Button
#define IDC_BTCANCEL 1035//Cancel Button
#define IDC_ETNEWVALUE 1036//Change Value
#define IDC_BTABOUT 1040//About Button
#define IDC_BTAT4RE 1041//AT4RE Button
#define IDC_BTEXIT 1042//Exit Button
#define IDC_BTINC 1043//Inc Button
#define IDC_BTDEC 1044//Dec Button
#define IDC_BTRESET 1045//Reset Button
 
HWND hWnd;
HANDLE hProcess;
DWORD ThreadId;
HINSTANCE hInstance;
HMENU hMenu;
HWND hListView;//查找内存地址结果列表
int bpsize;//硬件断点大小
PVOID* BaseAddrArray;//内存区块基址数组，用于查找
DWORD* AddrSizeArray;//内存区块大小数组，用于查找
BYTE** ResultDataArray;//结果地址数组
LPVOID ValueAddr;//要改写数据的地址
char String[256];//临时数组
bool IsHex,StopFind;//是否为16进制；是否停止搜索
int ToSearch;//要搜索的数据
 
DWORD WINAPI CreateMainDialog(LPVOID lpParameter)
{
    InitCommonControls();
    return DialogBoxParam(hInstance,MAKEINTRESOURCE(IDD_MAINDLG),NULL,MainDialogFunc,0);//打开主窗口
}
 
void WINAPI ListViewAddItem(DWORD addr,DWORD tosearch)
{
    LVITEM item;
    item.mask=LVIF_TEXT;
    item.iItem=SendMessage(hListView,LVM_GETITEMCOUNT,0,0);//获取当前项数目作为下次加入项的序号
    item.iSubItem=0;
    RtlZeroMemory(String, 256);
    wsprintf(String,"%.8X",(DWORD)BaseAddrArray[i]+j);
    item.pszText=String;
    SendMessage(hListView,LVM_INSERTITEM,0,(LPARAM)&item);//加入该项第一列地址
    item.mask=LVIF_TEXT;
    item.iSubItem++;
    RtlZeroMemory(String, 256);
    wsprintf(String,"%d",ToSearch);
    item.pszText=String;
    SendMessage(hListView,LVM_SETITEM,0,(LPARAM)&item);//加入该项第二列数据
}
 
DWORD WINAPI FindFirstThread(LPVOID lpParameter)//首次查找所启动的线程
{
    HWND hWnd=(HWND)lpParameter;
    RtlZeroMemory(String, 256);
    SendDlgItemMessage(hWnd,IDC_LVLIST,LVM_DELETEALLITEMS,0,0);
    //获取输入框内目标整数
    if(IsHex)
    {
        GetDlgItemText(hWnd,IDC_ETVAL,String,9);
        ToSearch=GetAddressFromString(String);
    }
    else
    {
        BOOL Translated;
        ToSearch=GetDlgItemInt(hWnd,IDC_ETVAL,&Translated,TRUE);
        if(!Translated)
            return MessageBox(hWnd,"Error occurred ","Cheat Utility Plugin",MB_TOPMOST|MB_ICONHAND);
    }
    //在各个内存区快中寻找目标数字
    for(int i=0;AddrSizeArray[i] && !StopFind;i++)
    {
        RtlZeroMemory(ResultDataArray,0x1000000);
        RtlZeroMemory(String, 256);
        int itemcount=SendDlgItemMessage(hWnd,IDC_LVLIST,LVM_GETITEMCOUNT,0,0);
        wsprintf(String,"Scanning : %.8X || %d item(s) found",BaseAddrArray[i],itemcount);
        SetDlgItemText(hWnd,IDC_SBSTATU,String);
        static DWORD NumberOfBytesRead;
        ReadProcessMemory(hProcess,BaseAddrArray[i],ResultDataArray,AddrSizeArray[i],&NumberOfBytesRead);
        for(int j=0;j<=AddrSizeArray[i] && !StopFind;j++)
        {
            DWORD data1=0,data2=0;
            if(bpsize == 4)
            {
                data1=ToSearch;
                data2=*(DWORD*)(ResultDataArray+j);
            }
            else if(bpsize == 2)
            {
                data1=ToSearch;
                data2=*(WORD*)(ResultDataArray+j);
            }
            else
            {
                data1=ToSearch;
                data2=*(BYTE*)(ResultDataArray+j);
            }
            if(data1 == data2)
                ListViewAddItem((DWORD)BaseAddrArray[i]+j,ToSearch);
        }
    }
    return 0;
}
 
DWORD WINAPI FindNextThread(LPVOID lpParameter)//再次搜索所启动的线程
{
    RtlZeroMemory(ResultDataArray,0x1000000);
    if(IsHex)
    {
        GetDlgItemText(hWnd,IDC_ETVAL,String,9);
        ToSearch=GetAddressFromString(String);
    }
    else
    {
        BOOL Translated;
        ToSearch=GetDlgItemInt(hWnd,IDC_ETVAL,&Translated,TRUE);
        if(!Translated)
        {
            MessageBox(hWnd,"Error occurred ","Cheat Utility Plugin",MB_TOPMOST|MB_ICONHAND);
            return 0;
        }
    }
    int resultaddrcount=0;
    int itemcount=SendDlgItemMessage(hWnd,IDC_LVLIST,LVM_GETITEMCOUNT,0,0);
    if(itemcount && itemcount != -1)
    {
        while(!StopFind && itemcount)
        {
            itemcount--;
            ListViewGetItem(itemcount,0);
            ulong addr=GetAddressFromString(String);
            RtlZeroMemory(String, 256);
            wsprintf(String,"Scanning : %.8X",addr);
            SetDlgItemText(hWnd,IDC_SBSTATU,String);
            DWORD data;
            ReadProcessMemory(hProcess,addr,&data,bpsize,NULL);
            if(data == ToSearch)
            {
                ResultDataArray[resultaddrcount]=addr;
                resultaddrcount++;
            }
        }
        if(itemcount)
            return 0;
        SetDlgItemText(hWnd,IDC_SBSTATU,"Generating List ...");
        SendDlgItemMessage(hWnd,IDC_LVLIST,LVM_DELETEALLITEMS,0,0);
        while(resultaddrcount--)
        {
            ListViewAddItem((DWORD)ResultDataArray[resultaddrcount],ToSearch);
        }
    }
    RtlZeroMemory(String, 256);
    itemcount=SendDlgItemMessage(hWnd,IDC_LVLIST,LVM_GETITEMCOUNT,0,0);
    wsprintf(String,"%d item(s) found",itemcount);
    SetDlgItemText(hWnd,IDC_SBSTATU,String);
    return 0;
}
 
extc void cdecl ODBG_Pluginaction(int origin,int action,void *item)
{
    if(origin != PM_MAIN)
        return;
    if(action == 0)
    {
        hProcess=(HANDLE)Plugingetvalue(VAL_HPROCESS);
        if(!hProcess)//未调试程序时不能打开窗口
            MessageBox(hWnd,"No Debugee loaded ","Cheat Utility Plugin",MB_TOPMOST|
MB_ICONEXCLAMATION);
        else
            CreateThread(NULL,0,CreateMainDialog,NULL,0,&ThreadId);//这里其实也可以不用多线程
    }
    else if(action == 1)//关于信息
        MessageBox(hWnd,"Cheat Utility Plugin v1.0\r\nCopyright (C) 2007 by GamingMasteR-AT4RE",
            "Cheat Utility Plugin",MB_ICONASTERISK);
}
 
extc int cdecl ODBG_Plugindata(char shortname[32])
{
    lstrcpy(shortname,"Cheat Utility");
    return 108;
}
 
extc int cdecl ODBG_Pluginmenu(int origin,char data[4096],void *item)
{
    if(origin != PM_MAIN)
        return 0;
    lstrcpy(data,"0 &Start|1 &About");
    return 1;
}
 
extc int cdecl ODBG_Plugininit(int ollydbgversion,HWND hw,ulong *features)
{
    if(ollydbgversion < 108)
        return -1;
    Addtolist(0,0,"Cheat Utility Plugin v1.0");
    hWnd=hw;
    return 0;
}
 
void WINAPI ListViewGetItem(int iItem,int iSubItem)
{//获取该项数据到String中
    LVITEM itemdata;
    itemdata.iItem=iItem;
    itemdata.iSubItem=iSubItem;
    itemdata.mask=LVIF_TEXT;
    itemdata.pszText=String;
    itemdata.cchTextMax=256;
    SendMessage(hListView,LVM_GETITEM,0,(LPARAM)&itemdata);
}
 
ulong GetAddressFromString(char* str)
{//作者秀了一下16进制字符串转整数的算法
    int len=strlen(str);
    ulong addr=0;
    int x1,x2=0;
    for(int i=0;i<len;i++)
    {
        if(str[i] < 'A')//0-9
        {
            x1=str[i]-'0';
        }
        else//A-F
        {
            x2=32*((str[i]<'W')+x2);
            x1=x2+str[i]-'W';
        }
        addr += ((x1&0xF)<<(4*i-1));
    }
    return addr;
}
 
int WINAPI SetDialogFunc(HWND hwndDlg,UINT uMsg,WPARAM wParam,LPARAM lParam)
{//更改数据窗口
    switch (uMsg)
    {
    case WM_INITDIALOG:
        break;
    case WM_COMMAND:
        if(wParam == IDC_BTOK)
        {
            ulong addr;
            if(IsHex)//如果为16进制数
            {
                GetDlgItemText(hwndDlg,IDC_ETNEWVALUE,String,9);
                addr=GetAddressFromString(String);
            }
            else
            {
                BOOL Translated;
                addr=GetDlgItemInt(hwndDlg,IDC_ETNEWVALUE,&Translated,TRUE);
                if(!Translated)
                {
                    MessageBox(hwndDlg,"Error occurred ","Cheat Utility Plugin",MB_TOPMOST|MB_ICONHAND);
                    return 0;
                }
            }
            WriteProcessMemory(hProcess,ValueAddr,&addr,bpsize,NULL);//改写数据
        }
        else if(wParam == IDC_BTCANCEL)
            EndDialog(hwndDlg,0);
        break;
    case WM_CLOSE:
        EndDialog(hwndDlg,0);
        break;
    }
    return 0;
}
 
void WINAPI SearchFreeMemoryBlock(HANDLE hProcess)
{
    MEMORY_BASIC_INFORMATION meminfo;
    RtlZeroMemory(String, 256);
    RtlZeroMemory(BaseAddrArray,0x1000000);
    RtlZeroMemory(AddrSizeArray,0x1000000);
    static int blockindex=0;
    for(int i=0x00400000;i<0x70000000;i+=meminfo.RegionSize,blockindex++)
    {//exe基址一般是0x00400000，而0x80000000以上为系统领空
        VirtualQueryEx(hProcess,i,&meminfo,sizeof(MEMORY_BASIC_INFORMATION));
        if(meminfo.Protect && meminfo.Protect != PAGE_NOACCESS && meminfo.Protect != PAGE_EXECUTE &&
            meminfo.Protect != PAGE_NOCACHE)
        {
            if(meminfo.State != MEM_FREE && meminfo.Type != MEM_MAPPED)
            {
                BaseAddrArray[blockindex]=meminfo.BaseAddress;
                AddrSizeArray[blockindex]=meminfo.RegionSize;
            }
        }
    }
}
 
int  WINAPI MainDialogFunc(HWND hwndDlg,UINT uMsg,WPARAM wParam,LPARAM lParam)
{//该回调将过程驱动编程转换为事件驱动编程
    static DWORD firstthreadid,nextthreadid;
    switch(uMsg)
    {
    case WM_INITDIALOG:
        hMenu=CreatePopupMenu();
        AppendMenu(hMenu,MF_ENABLED,IDM_FOLLOW,"Follow in Dump");
        AppendMenu(hMenu,MF_ENABLED,IDM_HARDWARE,"Hardware Breakpoint");
        AppendMenu(hMenu,MF_SEPARATOR,0,NULL);
        AppendMenu(hMenu,MF_ENABLED,IDM_CHANGE,"Change Value");
        AppendMenu(hMenu,MF_SEPARATOR,0,NULL);
        AppendMenu(hMenu,MF_ENABLED,IDM_DELETE,"Delete");
        //以下几行代码对列表框控件添加2列表头
        hListView=GetDlgItem(hwndDlg,1001);
        LVCOLUMN coldata;
        coldata.mask=LVCF_WIDTH|LVCF_TEXT;
        coldata.pszText="Address";
        coldata.cx=100;
        SendMessage(hListView,LVM_INSERTCOLUMN,0,(LPARAM)&coldata);
        coldata.pszText="Value";
        coldata.cx=100;
        SendMessage(hListView,LVM_INSERTCOLUMN,1,(WPARAM)&coldata);
        //默认数据（断点）为双字大小
        CheckDlgButton(hwndDlg,IDC_RBDWORD,BST_CHECKED);
        bpsize=sizeof(DWORD);
        BaseAddrArray=(PVOID*)VirtualAlloc(NULL,0x1000000,MEM_COMMIT,PAGE_READWRITE);
        VirtualLock(BaseAddrArray,0x1000000);
        ResultDataArray=(BYTE**)VirtualAlloc(NULL,0x1000000,MEM_COMMIT,PAGE_READWRITE);
        VirtualLock(ResultDataArray,0x1000000);
        AddrSizeArray=(DWORD*)VirtualAlloc(NULL,0x1000000,MEM_COMMIT,PAGE_READWRITE);
        VirtualLock(AddrSizeArray,0x1000000);
        break;
    case WM_COMMAND:
        if(lParam == 0)//选中菜单
        {
            switch(LOWORD(wParam))
            {//IDM_*
            case IDM_FOLLOW://跟随该地址
                {
                    int index=SendDlgItemMessage(hwndDlg,IDC_LVLIST,LVM_GETNEXTITEM,
                        -1,LVNI_FOCUSED);//找到第一个选中项
                    //这里处理的不好，居然是通过消息获取字符串再转换成地址，还不如事先存在数据结构中
                    ListViewGetItem(index,0);
                    Setcpu(0,0,GetAddressFromString(String),0,CPU_DUMPFIRST|CPU_DUMPFOCUS);
                    RtlZeroMemory(String,256);
                }
                break;
            case IDM_HARDWARE://在该地址处下硬件断点
                {
                    int index=SendDlgItemMessage(hwndDlg,IDC_LVLIST,LVM_GETNEXTITEM,
                        -1,LVNI_FOCUSED);//找到第一个选中项
                    ListViewGetItem(index,0);
                    Sethardwarebreakpoint(GetAddressFromString(String),bpsize,HB_ACCESS);
                    RtlZeroMemory(String,256);
                }
                break;
            case IDM_CHANGE://修改该地址数据
                {
                    int index=SendDlgItemMessage(hwndDlg,IDC_LVLIST,LVM_GETNEXTITEM,
                        -1,LVNI_FOCUSED);//找到第一个选中项
                    ListViewGetItem(index,0);
                    ValueAddr=(LPVOID)GetAddressFromString(String);
                    RtlZeroMemory(String,256);
                    DialogBoxParam(hInstance,MAKEINTRESOURCE(IDD_SETDIALOG),hwndDlg,
SetDialogFunc,0);
                }
                break;
            case IDM_DELETE://删除该结果项
                {
                    int index=SendDlgItemMessage(hwndDlg,IDC_LVLIST,LVM_GETNEXTITEM,
                        -1,LVNI_FOCUSED);//找到第一个选中项
                    SendDlgItemMessage(hwndDlg,IDC_LVLIST,LVM_DELETEITEM,index,0);
                }
                break;
            }
        }
        switch(wParam)
        {
        case IDC_BTFIRST://首次查找
            if(IsDlgButtonChecked(hwndDlg,IDC_RBBYTE))
                bpsize=sizeof(BYTE);
            else if(IsDlgButtonChecked(hwndDlg,IDC_RBWORD))
                bpsize=sizeof(WORD);
            else
                bpsize=sizeof(DWORD);
            if(IsDlgButtonChecked(hwndDlg,IDC_CBISHEX))
                IsHex=true;
            else
                IsHex=false;
            StopFind=false;
            SearchFreeMemoryBlock(hProcess);
            CreateThread(NULL,0,FindFirstThread,hwndDlg,0,&firstthreadid);//这里应该存储句柄以待后用
            break;
        case IDC_BTNEXT://再次查找
            StopFind=false;
            CreateThread(NULL,0,FindNextThread,hwndDlg,0,&nextthreadid);//这里应该存储句柄以待后用
            break;
        case IDC_BTSTOP://停止查找
            StopFind=true;
            break;
        case IDC_BTINC://自增数据
            {
                ulong addr;
                if(IsHex)
                {
                    GetDlgItemText(hwndDlg,IDC_ETNEWVALUE,String,9);//这句有误，控件ID作者搞错了
                    addr=GetAddressFromString(String);
                    RtlZeroMemory(String, 256);
                    wsprintf(String,"%.8X",addr+1);
                    SetDlgItemText(hwndDlg,IDC_ETVAL,String);
                }
                else
                {
                    BOOL Translated;
                    addr=GetDlgItemInt(hwndDlg,IDC_ETVAL,&Translated,TRUE);
                    if(!Translated)
                        MessageBox(hwndDlg,"Error occurred ","Cheat Utility Plugin",MB_TOPMOST|
MB_ICONHAND);
                    else
                        SetDlgItemInt(hwndDlg,IDC_ETVAL,addr+1,TRUE);
                }
            }
            break;
        case IDC_BTDEC://自减数据
            {
                ulong addr;
                if(IsHex)
                {
                    GetDlgItemText(hwndDlg,IDC_ETNEWVALUE,String,9);//这句有误，该控件不在主窗口中
                    addr=GetAddressFromString(String);
                    RtlZeroMemory(String, 256);
                    wsprintf(String,"%.8X",addr-1);
                    SetDlgItemText(hwndDlg,IDC_ETVAL,String);
                }
                else
                {
                    BOOL Translated;
                    addr=GetDlgItemInt(hwndDlg,IDC_ETVAL,&Translated,TRUE);
                    if(!Translated)
                        MessageBox(hwndDlg,"Error occurred ","Cheat Utility Plugin",MB_TOPMOST|
MB_ICONHAND);
                    else
                        SetDlgItemInt(hwndDlg,IDC_ETVAL,addr-1,TRUE);
                }
            }
            break;
        case IDC_BTRESET://重新初始化数据
            CheckDlgButton(hwndDlg,IDC_RBDWORD,BST_CHECKED);
            CheckDlgButton(hwndDlg,IDC_CBISHEX,BST_UNCHECKED);
            SetDlgItemInt(hwndDlg,IDC_ETVAL,0,TRUE);
            SendDlgItemMessage(hwndDlg,IDC_LVLIST,LVM_DELETEALLITEMS,0,0);
            bpsize=sizeof(DWORD);
            IsHex=false;
            RtlZeroMemory(BaseAddrArray,0x1000000);
            RtlZeroMemory(ResultDataArray,0x1000000);
            RtlZeroMemory(String,256);
            break;
        case IDC_BTABOUT:
            MessageBox(hWnd,"Cheat Utility Plugin v1.0\r\nCopyright (C) 2007 by GamingMasteR-AT4RE",
                "Cheat Utility Plugin",MB_ICONASTERISK);
            break;
        case IDC_BTAT4RE:
            ShellExecute(NULL,"Open","http://www.at4re.com/",NULL,NULL,SW_MAXIMIZE);
            break;
        case IDC_BTEXIT:
            SendMessage(hwndDlg,WM_CLOSE,0,0);
            break;
        }
        break;
    case WM_NOTIFY:
        {
            NMHDR* nmhdr=(NMHDR*)lParam;
            if(nmhdr->hwndFrom == hListView)
            {
                if(nmhdr->code == NM_DBLCLK)//双击项目则打开修改数据对话框
                {
                    int index=SendDlgItemMessage(hwndDlg,IDC_LVLIST,LVM_GETNEXTITEM,
                        -1,LVNI_FOCUSED);//找到第一个选中项
                    ListViewGetItem(index,0);
                    ValueAddr=(LPVOID)GetAddressFromString(String);
                    RtlZeroMemory(String, 256);
                    DialogBoxParam(hInstance,MAKEINTRESOURCE(IDD_SETDIALOG),hwndDlg,SetDialogFunc,0);
                }
                else if(nmhdr->code == NM_RCLICK)//右击项目
                {
                    static POINT Point;
                    GetCursorPos(&Point);
                    TrackPopupMenu(hMenu,TPM_LEFTALIGN|TPM_TOPALIGN|TPM_LEFTBUTTON,
Point.x,Point.y,0,hwndDlg,NULL);
                }
            }
        }
        break;
    case WM_CLOSE:
        CloseHandle(firstthreadid);//错误用法
        VirtualFree(ResultDataArray,0x1000000,MEM_DECOMMIT);
        VirtualFree(BaseAddrArray,0x1000000,MEM_DECOMMIT);
        VirtualFree(AddrSizeArray,0x1000000,MEM_DECOMMIT);
        EndDialog(hwndDlg,0);
        break;
    }
    return 0;
}
```

### 总结
&emsp;&emsp;插件确实有用类似于CheatEngine修改数据，鉴于CheatEngine开源，有时间我会加以完善，同时在逆向过程中也发现了一些不足之处，如下：
* 1.只支持整数
* 2.有些逻辑不够全面，例如IsHex选中的处理
* 3.我逆向的结果证明作者源码中存在一些小错误，例如Inc/Dec按钮消息和WM_CLOSE消息处理
* 4.16进制转换代码是有函数库可以完成，且更完善易用
* 5.某些数据应通过数据结构存储，而不是通过API函数发送消息再获得，这样效率低

## 第五章 OllyDbg去除花指令插件DeJunk源码分析

&emsp;&emsp;该插件为花指令自动去除插件。经多人完善，最初作者为ljtt，本版为flyfancy根据hoto制作的dejunk插件修改而来。文件大小29kb，代码段占4.8kb，逆向代码大约500行。我也是通过逆向分析才第一次了解到花指令是什么以及如何去除花指令。先做个简要介绍：

```C++
+0 ;        jo        label
+2 ;        jno        label
+4 ;        db        _junkcode
+5 ;label:        ....
;                ....
```

&emsp;&emsp;上面的汇编代码，左边序号暗示了指令所占字节数，jo和jno都调到+5处，因此，前5字节是无效的，相当于直接执行+5处代码，而这5字节就是为了迷惑反汇编器而构造的汇编指令，常见的修改方法是把前5字节都用nop指令代替即可。该例比较简单，然而如果作者精心构造的复杂花指令可以很复杂，很大程度提升反汇编分析时间。

### 逆向

```C++
#include <windows.h>
#include <commctrl.h>
#include "Plugin.h"
 
#define IDC_BTClOSE 8//Close按钮
#define IDD_MAINDLG 101//主对话框
#define IDC_BTSTART 1000//Start按钮
#define IDC_ETSTARTADDR 1001//Start Addr编辑框
#define IDC_ETRANG 1002//Rang编辑框
#define IDC_CBDIRECTION 1003 //Direction选择框
#define IDC_SBSTATU 1004//状态栏
#define IDC_CBJUNKTYPE 1005//Junk Type选择框
#define IDC_BTE 1006//E按钮
HINSTANCE hInstance;
char JunkdbcfgPath[260],DeJunkLogPath[260],String2[260];
HWND hOllyWnd;//OllyDbg窗口句柄
char classname[32];//插件主窗口类名
HWND hStartAddr,hRang,hDirection,hJunkType,hStart,hClose,hE;//和资源对应
char String1[9];//临时存储
ulong JunkStartAddr,JunkRange;
ulong JunkCodeNum;
bool IsDefaultType;
bool IsMallocSuccess;//是否成功分配了空间
void* JunkUndoData=NULL,*JunkData=NULL;
HANDLE hLogFile;
BOOL WINAPI DllMain(HINSTANCE hinstDLL,DWORD fdwReason,LPVOID lpvvReserved)
{
    if(fdwReason == DLL_PROCESS_ATTACH)
    {
        hInstance=hinstDLL;
        GetModuleFileName(hinstDLL,,260);
        strrchr(JunkdbcfgPath,'\\')[1]='\0';//便于连接成文件路径
        strcpy(DeJunkLogPath,JunkdbcfgPath);
        strcat(JunkdbcfgPath,"Junkdb.cfg");
        strcat(DeJunkLogPath,"DeJunk.Log");
    }
}
LRESULT WINAPI MainProc(HWND hWnd,UINT uMsg,WPARAM wParam,LPARAM lParam)
{
    switch (uMsg)
    {
        case WM_DESTROY:
        case WM_SETFOCUS:
        case WM_PAINT:
            break;
        default:
            return DefWindowProc(hWnd,uMsg,wParam,lParam);
    }
    return 0;
}
BOOL IsInputValid(HWND hDlg,int nID)
{//检测输入值是否为合法16进制数
    int len=SendDlgItemMessage(hDlg,nID,WM_GETTEXTLENGTH,0,0);
    GetDlgItemText(hDlg,nID,String1,9);
    for(int i=0;i<len;i++)
    {
        if(!isxdigit(String1[i]))
            return FALSE;
    }
    return TRUE;
}
ulong GetNumberFromString(char* str)
{//和上一个插件例子是同一个函数，将字符串转16进制数，可见是一个作者
    int len=strlen(str);
    ulong addr=0;
    int x1,x2=0;
    for(int i=0;i<len;i++)
    {
        if(str[i] < 'A')//0-9
        {
            x1=str[i]-'0';
        }
        else//A-F
        {
            x2=32*((str[i]<'W')+x2);
            x1=x2+str[i]-'W';
        }
        addr += ((x1&0xF)<<(4*i-1));
    }
    return addr;
}
/*---------Junkdb.cfg-----------
    ; default value
    DefaultRang=01000
    DefaultType=Custom
    EnableLog=1
 
    ;set JunkType combol list
    JunkType=Common,TELock,UltraProtect,Custom
*/
void InitComboBox(HWND hWnd)
{//从配置文件读取类型，并添加到花指令类型列表框
    char JunkTypeStr[260],content[260];
    memset(JunkTypeStr,0,260);
    GetPrivateProfileString("OPTION","JunkType",NULL,JunkTypeStr,260,JunkdbcfgPath);
    char* ptr=JunkTypeStr;
    while(ptr[0] != '\0')
    {//拆分字符串 string1,string2,string3,...
        char* pos1=strchr(ptr,',');
        if(pos1)
        {
            *pos1='\0';
            strcpy(content,ptr);
            ptr=pos1+1;
        }
        else
            strcpy(content,ptr);
        SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_ADDSTRING,0,(LPARAM)content);
        if(!pos1)
        {
            SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_SETCURSEL,0,0);
            break;
        }
    }
    if(!GetPrivateProfileString("OPTION","DefaultRang",NULL,String1,6,JunkdbcfgPath))
        wsprintf(String1,"%05lX",4096);//默认搜索区块大小
    SetDlgItemText(hWnd,IDC_ETRANG,String1);
    if(!GetPrivateProfileString("OPTION","DefaultType",NULL,String2,260,JunkdbcfgPath))
        SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_SETCURSEL,0,0);
    else
        SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_SELECTSTRING,0,(LPARAM)String2);
}
void Transform(uchar* Dest,const char* Source,int len)
{//16进制字符串按2位转换成字节码数据
    char d[3];
    int curnum;
    memset(Dest,0,260);
    if(len <= 0)
        return;
    do
    {
        strncpy(d,Source,2);
        d[2]='\0';
        Source+=2;
        curnum=GetNumberFromString(d);
        if(d[0] == '?')
            *Dest=0x90;
        else
            *Dest=curnum;
        Dest++;
    } 
    while (--len);
}
void FindMatch(uchar* data,BYTE* S,BYTE* R,int len,int Range)
{//查找目的范围内匹配的花指令，本程序核心算法之所在
    char buf[260];
    DWORD writenum;
    uchar* ptr=data;
    int i,j;
    while(true)
    {
        i=ptr-data;
        if(S[0] != 0x90)
            ptr=(uchar*)memchr(ptr,S[0],Range-i);
        if(!ptr || Range-i<len)//找不到花指令
            break;
        for(j=0;j<len;j++)
        {
            if(S[j] != ptr[j] && S[j] != 0x90)
            {//原S串的'?'代表任意1个16进制数，经过Transform函数处理成0x90，因此如果遇到S串为0x90，则为统配符，直接跳过，否则要进行对比
                ptr++;
                break;
            }
        }
        if(j == len-1)
        {//长度匹配因此找到一个花指令
            memcpy(ptr,R,len);//写入对应的R串以去除花指令并能正常运行
            wsprintf(buf,"\t0x%08lX\r\n",JunkStartAddr+i);
            WriteFile(hLogFile,buf,strlen(buf),&writenum,NULL);
            ptr += len;
            JunkCodeNum++;
        }
    }
}
void FindJunk(HWND hWnd,ulong StartAddr,ulong Range)
{//查找花指令
    char buf[260];
    DWORD writenum;
    static SYSTEMTIME SystemTime;
    char PrePatStr[512],JunkTypeStr[512];
    char SearchType[260],JunkTypeName[260],SectionName[260];
    int Slen,Rlen;
    char Soriserial[512],Roriserial[512];
    uchar Snewserial[260],Rnewserial[260];
    if(Range == 0)
    {
        MessageBox(hWnd,"Please give deJunk rang!","Warning",MB_OK|MB_ICONEXCLAMATION);
        return;
    }
    hLogFile=CreateFile(DeJunkLogPath,GENERIC_READ|GENERIC_WRITE,FILE_SHARE_READ|FILE_SHARE_WRITE,
        NULL,CREATE_ALWAYS,FILE_ATTRIBUTE_NORMAL,NULL);
    JunkData=malloc(Range);
    JunkCodeNum=0;
    if(!JunkData)
    {
        MessageBox(hWnd,"Can't allocates memory blocks!","Warning",MB_OK|MB_ICONEXCLAMATION);
        return;
    }
    if(!Readmemory(JunkData,StartAddr,Range,MM_SILENT))//读取程序目标地址字节码
    {
        DWORD writenum;
        MessageBox(hWnd,"Can't read the memory space!","Warning",MB_OK|MB_ICONEXCLAMATION);
        goto RET;
    }
    if(IsDefaultType)
    {
        GetPrivateProfileString("OPTION","DefaultType",0,SearchType,260,JunkdbcfgPath);
        IsDefaultType=false;
    }
    else
    {
        int index=SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_GETCURSEL,0,0);
        SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_GETLBTEXT,index,(LPARAM)SearchType);
    }
    strcpy(JunkTypeName,"PatList_");
    strcat(JunkTypeName,SearchType);
    GetLocalTime(&SystemTime);
    wsprintf(buf,"[-= DeJunk Last Log =-]\r\n\r\nLog Time: %04d-%02d-%02d  %02d:%02d:%02d\r\n\r\nSearch Address:\r\n"
        "\t0x%08lX ~ 0x%08lX\r\n\r\nSearch Type:\r\n\t%s\r\n\r\nFind junk code in:\r\n",SystemTime.wYear,SystemTime.wMonth,
        SystemTime.wDay,SystemTime.wHour,SystemTime.wMinute,SystemTime.wSecond,JunkStartAddr,
        JunkStartAddr+JunkRange,SearchType);
    WriteFile(hLogFile,buf,strlen(buf),&writenum,NULL);
    GetPrivateProfileString("OPTION","PrePatName",NULL,PrePatStr,512,JunkdbcfgPath);
    GetPrivateProfileString("OPTION",JunkTypeName,NULL,JunkTypeStr,512,JunkdbcfgPath);
    char* ptr=JunkTypeStr,*pos1=NULL;
    while(pos1 && *ptr)
    {//对该模式下所有花指令类型，作如下操作
/*      模式列表几对应花指令集合：
    ---------Junkdb.cfg-----------
    PatList_Common=_T1,_T2,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22
    PatList_TELock=_jmp02,_jnz01,_jmp01,_telock_call02_1,_telock_call02_2,_slc_jb01,_slc_jb02,_clc_jnb01,_clc_jnb02
    PatList_UltraProtect=1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,_jmp01,_jmp11,_jmp12,_jmp13,_jmp15,_call01,_call011,_call012
    PatList_Custom=_jmp02,_jnz01,_jmp01
*/
        pos1=strchr(ptr,',');
        strcpy(SectionName,PrePatStr);
        if(pos1)
        {
            *pos1='\0';
            strcat(SectionName,ptr);
            ptr=pos1+1;
        }
        else
            strcat(SectionName,ptr);
        Slen=GetPrivateProfileString(SectionName,"S",NULL,Soriserial,512,JunkdbcfgPath);//S串为识别出的花指令序列模式
        Rlen=GetPrivateProfileString(SectionName,"R",NULL,Roriserial,512,JunkdbcfgPath);//R串为对应S串的正常指令序列模式
/*    例如Custom模式：
---------Junkdb.cfg-----------
    [CODE_jnz01]
    S = 7501??
    R = 909090
 
    [CODE_jmp01]
 
    S = EB01??
    R = 909090
 
    [CODE_jmp02]
 
    S = EB02????
    R = 90909090
*/
        if(Slen !=Rlen || !Slen)//S串的长度应该和R匹配，否则无法正常替换
        {
            wsprintf(buf,"Junkdb file [%s] section read error.",SectionName);
            MessageBox(hWnd,buf,"DeJunk plugin v0.12",MB_OK|MB_ICONASTERISK);
            free(JunkUndoData);
            JunkUndoData=NULL;
            return;
        }
        Transform(Snewserial,Soriserial,Slen/2);//字符串转换为字节码
        Transform(Rnewserial,Roriserial,Rlen/2);
        FindMatch((uchar*)JunkData,Snewserial,Rnewserial,Slen/2,Range);//查找并替换花指令为正常代码
    }
    if(JunkCodeNum == 0)
    {
        wsprintf(buf,"Cannot find Junk code.");
        MessageBox(hWnd,buf,"DeJunk plugin v0.12",MB_OK|MB_ICONASTERISK);
    }
    else//如果找到则需要把原始字节码保存以便恢复
    {
        if(JunkUndoData)
        {
            free(JunkUndoData);
            JunkUndoData=NULL;
        }
        JunkUndoData=malloc(Range);
        if(!JunkUndoData)
            MessageBox(hWnd,"Can't allocates Undo memory blocks!\n  Undo function invalid.",
                "Warning",MB_OK|MB_ICONEXCLAMATION);
        IsMallocSuccess=true;
        if(!Readmemory(JunkUndoData,StartAddr,Range,MM_SILENT))//将原始字节码保存在JunkUndoData中
            MessageBox(hWnd,"Can't read the Undo data!","Warning",MB_OK|MB_ICONEXCLAMATION);
        if(!Writememory(JunkData,StartAddr,Range,MM_SILENT))//将修改后的字节码覆盖内存中原始代码段
            MessageBox(hWnd,"Can't Write the memory space!","Warning",MB_OK|MB_ICONEXCLAMATION);
        wsprintf(buf,"%d junk code were replace.",JunkCodeNum);
        MessageBox(hWnd,buf,"DeJunk plugin v0.12",MB_OK|MB_ICONASTERISK);
        Setdisasm(JunkStartAddr,JunkRange,CPU_ASMFOCUS);//将修改的结果在反汇编窗口显示
    }
RET:
    WriteFile(hLogFile,"\r\n",2,&writenum,NULL);
    WriteFile(hLogFile,buf,260,&writenum,NULL);
    CloseHandle(hLogFile);
    free(JunkData);
    JunkData=NULL;
}
int WINAPI DeJunkFunc(HWND hWnd,UINT uMsg,WPARAM wParam,LPARAM lParam)
{//去除花指令窗口界面
    switch (uMsg)
    {
        case WM_CLOSE:
            EndDialog(hWnd,0);
            break;
        case WM_SETCURSOR:
            {
                char* str;
                if( (HWND)wParam == hStartAddr)
                    str="DeJunk start address.(hex)";
                else if((HWND)wParam == hRang)
                    str="DeJunk rang.(hex)";
                else if((HWND)wParam == hDirection)
                    str="Select Direction.";
                else if((HWND)wParam == hJunkType)
                    str="Select Dejunk Type.";
                else if((HWND)wParam == hStart)
                        str="Start DeJunk.";
                else if((HWND)wParam == hE)
                        str="Edit Dejunk profile.";
                else if((HWND)wParam == hClose)
                        str="End Dialog.";
                else
                        str="Ready.";
                SendDlgItemMessage(hWnd,IDC_SBSTATU,WM_SETTEXT,0,(LPARAM)str);
            }
            break;
        case WM_INITDIALOG://初始化
            SetWindowText(hWnd,"DeJunk plugin v0.12");
            hStartAddr=GetDlgItem(hWnd,IDC_ETSTARTADDR);
            hRang=GetDlgItem(hWnd,IDC_ETRANG);
            hDirection=GetDlgItem(hWnd,IDC_CBDIRECTION);
            hJunkType=GetDlgItem(hWnd,IDC_CBJUNKTYPE);
            hStart=GetDlgItem(hWnd,IDC_BTSTART);
            hClose=GetDlgItem(hWnd,IDC_BTClOSE);
            hE=GetDlgItem(hWnd,IDC_BTE);
            PostMessage(hStartAddr,EM_SETLIMITTEXT,8,0);
            PostMessage(hRang,EM_SETLIMITTEXT,5,0);
            ulong curthreadid=Getcputhreadid();
            if(curthreadid)
                wsprintf(String1,"%08lX",Findthread(curthreadid)->reg.ip);//EIP
            else
                wsprintf(String1,"%08lX",0);
            SetDlgItemText(hWnd,IDC_ETSTARTADDR,String1);//起始地址设置为当前EIP
            SendDlgItemMessage(hWnd,IDC_CBDIRECTION,CB_ADDSTRING,0,(LPARAM)"Down");
            SendDlgItemMessage(hWnd,IDC_CBDIRECTION,CB_ADDSTRING,0,(LPARAM)"Up");
            SendDlgItemMessage(hWnd,IDC_CBDIRECTION,CB_SETCURSEL,0,0);
            SendDlgItemMessage(hWnd,IDC_SBSTATU,WM_SETTEXT,0,(LPARAM)"Ready.");
            InitComboBox(hWnd);
            break;
        case WM_COMMAND:
            switch(wParam)
            {
                case IDC_BTClOSE:
                    SendMessage(hWnd,WM_CLOSE,0,0);
                    break;
                case IDC_BTSTART:
                    if(!IsInputValid(hWnd,IDC_ETSTARTADDR) || !IsInputValid(hWnd,IDC_ETRANG))//检查输入合法性
                    {
                        MessageBox(hWnd,"Please input HEXadecimal.","DeJunk plugin v0.12",MB_OK|MB_ICONEXCLAMATION);
                        return 0;
                    }
                    //从输入获取数据
                    GetDlgItemText(hWnd,IDC_ETSTARTADDR,String1,9);
                    JunkStartAddr=GetNumberFromString(String1);
                    GetDlgItemText(hWnd,IDC_ETRANG,String1,0);
                    JunkRange=GetNumberFromString(String1);
                    if(SendDlgItemMessage(hWnd,IDC_CBDIRECTION,CB_GETCURSEL,0,0) == 1)//"Up"  如果向上查找则调整起始地址
                        JunkStartAddr -= JunkRange;
                    FindJunk(hWnd,JunkStartAddr,JunkRange);//找到并修改花指令
                    if(JunkCodeNum)
                        SendMessage(hWnd,WM_CLOSE,0,0);
                    break;
                case IDC_BTE://打开cfg配置文件进行编辑
                    GetPrivateProfileString("OPTION","Editor",NULL,String2,260,JunkdbcfgPath);
                    ShellExecute(NULL,"Open",String2,JunkdbcfgPath,NULL,SW_SHOWDEFAULT);
                    break;
                default:
                    break;
            }
        default:
            break;
    }
    return 0;
};
int WINAPI OptionFunc(HWND hWnd,UINT uMsg,WPARAM wParam,LPARAM lParam)
{//选项对话框窗口回调函数
    BOOL IsAccepted;
    switch(uMsg)
    {
        case WM_CLOSE:
            EndDialog(hWnd,0);
            return 0;
        case WM_SETCURSOR:
            {//鼠标移动
                char* str;
                if((HWND)wParam == hRang)
                    str="DeJunk rang.(hex)";
                else if((HWND)wParam == hJunkType)
                    str="Select Dejunk Type.";
                else if((HWND)wParam == hE)
                    str="Select default editor.";
                else if((HWND)wParam == hStart)
                    str="Save option";
                else if((HWND)wParam == hClose)
                    str="End Dialog.";
                else
                    str="Ready.";
                SendDlgItemMessage(hWnd,IDC_SBSTATU,WM_SETTEXT,0,(LPARAM)str);
            }
            break;
        case WM_INITDIALOG://初始化
            hRang=GetDlgItem(hWnd,IDC_ETRANG);
            hJunkType=GetDlgItem(hWnd,IDC_CBJUNKTYPE);
            hStart=GetDlgItem(hWnd,IDC_BTSTART);
            hClose=GetDlgItem(hWnd,IDC_BTClOSE);
            hE=GetDlgItem(hWnd,IDC_BTE);
            EnableWindow(GetDlgItem(hWnd,IDC_ETSTARTADDR),FALSE);
            EnableWindow(GetDlgItem(hWnd,IDC_CBDIRECTION),FALSE);
            SetWindowText(GetDlgItem(hWnd,IDC_BTSTART),"&Save");
            SetWindowText(hWnd,"Option");
            InitComboBox(hWnd);
            //读取配置文件
            if(!GetPrivateProfileString("OPTION","DefaultRang",NULL,String1,6,JunkdbcfgPath))
                wsprintf(String1,"%05lX",4096);
            SetDlgItemText(hWnd,IDC_ETRANG,String1);
            if(!GetPrivateProfileString("OPTION","DefaultType",NULL,String2,260,JunkdbcfgPath))
                SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_SETCURSEL,0,0);
            else
                SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_SELECTSTRING,0,(LPARAM)String2);
            break;
        case WM_COMMAND:
            if(wParam == IDC_BTClOSE)
                SendMessage(hWnd,WM_CLOSE,0,0);
            else if(wParam == IDC_BTSTART)
            {
                if(IsAccepted)
                    WritePrivateProfileString("OPTION","Editor",String2,JunkdbcfgPath);//设置cfg文件默认打开软件
                int index=SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_GETCURSEL,0,0);
                SendDlgItemMessage(hWnd,IDC_CBJUNKTYPE,CB_GETLBTEXT,index,(LPARAM)String2);
                WritePrivateProfileString("OPTION","DefaultType",String2,JunkdbcfgPath);
                if(IsInputValid(hWnd,IDC_ETRANG))
                {
                    GetDlgItemText(hWnd,IDC_ETRANG,String1,6);
                    WritePrivateProfileString("OPTION","DefaultRang",String1,JunkdbcfgPath);
                }
                SendMessage(hWnd,WM_CLOSE,0,0);
            }
            else if(wParam == IDC_BTE)
            {//选择cfg文件默认打开软件
                static OPENFILENAME filestruct;
                memset(String2,0,sizeof(String2));
                filestruct.hInstance=hInstance;
                filestruct.lStructSize=sizeof(OPENFILENAME);
                filestruct.nMaxFile=260;
                filestruct.lpstrFile=String2;
                filestruct.Flags=OFN_LONGNAMES|OFN_EXPLORER|OFN_FILEMUSTEXIST|OFN_PATHMUSTEXIST|OFN_HIDEREADONLY;
                filestruct.lpstrFilter="Exe File";
                IsAccepted=GetOpenFileName(&filestruct);
            }
            break;
        default:
            break;
    }
    return 0;
}
extc int  _export cdecl ODBG_Plugininit(int ollydbgversion,HWND hw,ulong *features)
{
    hOllyWnd=hw;
    if(Registerpluginclass(classname,NULL,hInstance,MainProc) < 0)
        return -1;
    char str[260];
    strcpy(str,"DeJunk plugin v0.12");
    strcat(str," by flyfancy");
    Addtolist(0,0,str);
    return 0;
}
extc int  _export cdecl ODBG_Plugindata(char shortname[32])
{
    strcpy(shortname,"DeJunk");
    return 108;
}
extc void _export cdecl ODBG_Plugindestroy(void)
{
    if(IsMallocSuccess)
    {
        free(JunkUndoData);
        JunkUndoData=NULL;
    }
    return Unregisterpluginclass(classname);
}
extc void _export cdecl ODBG_Pluginaction(int origin,int action,void *item)
{
    switch(action)
    {
        case 0://About
            MessageBox(hOllyWnd,"DeJunk plugin\n  Written by flyfancy\n  Thx: ljtt","DeJunk plugin v0.12",MB_OK|MB_ICONASTERISK);
            break;
        case 1://DeJunk
            DialogBoxParam(hInstance,MAKEINTRESOURCE(IDD_MAINDLG),hOllyWnd,DeJunkFunc,0);
            break;
        case 2://Undo
            if(!IsMallocSuccess)
            {
                MessageBox(hOllyWnd,"Undo data is NULL.","DeJunk plugin v0.12",MB_OK|MB_ICONASTERISK);
                return;
            }
            if(!Writememory(JunkUndoData,JunkStartAddr,JunkRange,MM_SILENT))
            {
                MessageBox(hOllyWnd,"Can't execute undo operation!","Warning",MB_OK|MB_ICONEXCLAMATION);
                return;
            }
            IsMallocSuccess=false;
            free(JunkUndoData);
            JunkUndoData=NULL;
            Setdisasm(JunkStartAddr,JunkRange,CPU_ASMFOCUS);
            break;
        case 3://Option
            DialogBoxParam(hInstance,MAKEINTRESOURCE(IDD_MAINDLG),hOllyWnd,OptionFunc,0);
            break;
        case 4://Dejunk selection
            if(origin == PM_DISASM && item != NULL)
            {
                t_dump* dump=(t_dump*)item;
                IsDefaultType=true;//设置当前JunkType为默认JunkType
                JunkStartAddr=dump->sel0;//根据反汇编窗口选择范围设置花指令范围
                JunkRange=dump->sel1 - JunkStartAddr;
                FindJunk(hOllyWnd,JunkStartAddr,JunkRange);
            }
            break;
        case 5://View last log
            GetPrivateProfileString("OPTION","Editor",NULL,String2,260,JunkdbcfgPath);
            ShellExecute(NULL,"Open",String2,JunkdbcfgPath,NULL,SW_SHOWDEFAULT);
            break;
    }
}
extc int  _export cdecl ODBG_Pluginmenu(int origin,char data[4096],void *item)
{
    if(origin != PM_MAIN)
        return 0;
    strcpy(data,"1 &DeJunk\tAlt+Shift+S, 2 &Undo\tAlt+Shift+Z, 4 D&ejunk selection\tAlt+Shift+Q|3 "\
        "&Option, 5 &View last log\tAlt+Shift+G|0 &About");
    return 1;
}
extc int  _export cdecl ODBG_Pluginshortcut(int origin,int ctrl,int alt,int shift,int key,void *item)
{
    if(ctrl == 0 && alt == 1 && shift == 1)
    {
        switch(key)
        {
            case 'S':
                ODBG_Pluginaction(PM_MAIN,1,NULL);//DeJunk
                return 1;
            case 'Q':
                if(origin == PM_DISASM && item != NULL)
                {
                    ODBG_Pluginaction(PM_DISASM,4,item);//Dejunk selection
                    return 1;
                }
                break;
            case 'G':
                ODBG_Pluginaction(PM_MAIN,5,NULL);//View last log
                return 1;
            case 'Z':
                ODBG_Pluginaction(PM_MAIN,2,NULL);//Undo
                return 1;
            default:
                break;
    }
    return 0;
}
```

### 实例

弄懂了代码来进行实例分析，目标源码如下：TestDejunk.asm  

```Asm
.386
.model flat,stdcall
option casemap:none
 
include windows.inc
include Masm32.inc
include kernel32.inc
include user32.inc
includelib kernel32.lib
includelib user32.lib
 
.data
szCap       db "Test",0
szFind      db "Debugger present",0
szNoFind    db "Debugger NOT present",0
szNote      db "Test Last Error Message",0
 
.code
start:
    ;test dejunk
    jmp @junk1
    db 075h
    db 001h
@junk1:
 
    ;test GetLastError
    invoke  MessageBox, NULL, addr szNote, addr szCap, MB_OK
    ;error usage function.
    invoke LoadIcon, NULL, NULL
    invoke SetWindowText, NULL, NULL
     
    ;test IsDebuggerPresent
    invoke  IsDebuggerPresent
    or eax, eax
    jne @Find
    invoke  MessageBox, NULL, addr szNoFind, addr szCap, MB_OK
@exit:
    invoke  ExitProcess, 0
@Find:
    invoke  MessageBox, NULL, addr szFind, addr szCap, MB_OK
    jmp @exit
end start
```

编译成exe放入OllyDbg调试，得到：

```Txt
00401000 >/$ EB 02          JMP SHORT TestDeju.00401004
00401002  |  75             DB 75                                    ;  CHAR 'u'
00401003  |  01             DB 01
00401004  |> 6A 00          PUSH 0                                   ; /Style = MB_OK|MB_APPLMODAL
```

对照Junkdb.cfg，可见对应于：

```Txt
[CODE_jmp02]
S = EB02????
R = 90909090
```

运行插件，果然将前4个字节改成了nop，跳过了。。。

## 第六章 OllyDbg导入IDA符号插件LoadMap分析

&emsp;&emsp;LoadMap插件可以在OllyDbg中调试程序时显示IDA符号，大大降低代码分析难度。先用IDA制作map文件，先用IDA载入exe文件，手动分析修改成容易理解的符号后，File->Produce file->Create map file创建map文件，使用OllyDbg载入exe文件，在插件界面选择map文件路径，就可以看到OllyDgb带符号了。
    经我分析，该插件代码并不复杂，本质上还是解析map文件，而该文件格式并不复杂，下面是我截取的一段：


|Start        |Length    |Name  |Class|
|-------------|----------|------|-----|
|0002:00000000|0000AF000H|.text |CODE|
|0003:00000000|00005B000H|.data |DATA|
|0004:00000000|000001000H|.tls  |DATA|
|0005:00000000|000001000H|.rdata|DATA|
|0006:00000000|000002000H|.idata|DATA|

|Address      |ics by Value|     
|-------------|------------|
|0001:00000000|start|
|0001:00000012|loc_6051012|
|0001:00000059|__GetExceptDLLinfo|
|0001:00000140|sub_6051140|
|0001:00000150|loc_6051150|
|0001:0000018D|loc_605118D|
|0001:0000018F|loc_605118F|
|0001:000001B1|loc_60511B1|
|0001:000001BE|loc_60511BE|
|0001:000001F1|loc_60511F1|
|0001:000044AC|_Assemble|
|0001:000044D3|loc_60554D3|
|0001:000044D8|loc_60554D8|
|0001:000044E6|loc_60554E6|
|0001:000044F8|loc_60554F8|
|0001:00004514|loc_6055514|

### 逆向

```C++
view sourceprint?
#include <Windows.h>
#include "Plugin.h"
#include <stdio.h>
 
HINSTANCE hInst;
HWND hOllyWnd;
t_module* MainModule;
BOOL WINAPI DllMain(HINSTANCE hinstDLL,DWORD fdwReason,LPVOID lpvReserved)
{
    if(fdwReason == DLL_PROCESS_ATTACH)
        hInst=hinstDLL;
    return 1;
}
 
extc int  _export cdecl ODBG_Pluginmenu(int origin,char data[4096],void *item)
{
    if(origin != PM_MAIN)
        return 0;
    memcpy(data,"0 载入 Map 文件|1 关于",25);
    return 1;
}
 
extc int  _export cdecl ODBG_Plugininit(int ollydbgversion,HWND hw, ulong *features)
{
    if(ollydbgversion < 108)
        return -1;
    hOllyWnd=hw;
    Addtolist(0,0,"LoadMap version 0.1 by lgx/iPB loaded");
    return 0;
}
 
extc int  _export cdecl ODBG_Plugindata(char shortname[32])
{
    strcpy(shortname,"LoadMap");
    return 108;
}
 
void LoadMap()
{
    ulong mainbase=Plugingetvalue(VAL_MAINBASE);
    if(!mainbase)
    {
        Addtolist(0,1,"MapConv 错误: 无进程用于添加 map 信息");
        MessageBox(hOllyWnd,"是这样 - 如果你没有调试任何程序 - 你不需要 .map 文件 ;-)","LoadMap v0.1",MB_OK|MB_ICONASTERISK);
        return;
    }
    MainModule=Findmodule(mainbase);
    Addtolist(0,0,"MainBase: %X, CodeBase: %X, %s",mainbase,MainModule->codebase,MainModule->path);
    char MapFileName[256];
    FILE* file;
    char str[256];
    MapFileName[0]= '\0';
    if(!Browsefilename("选择 map 文件",MapFileName,".map",0))
        return;
     file=fopen(MapFileName,"rt");
    if(!file)
    {
        Addtolist(0,1,"LoadMap 错误: 无法打开 %s",MapFileName);
        return;
    }
/*  map文件格式
     0001:00000000       start
     0001:00000012       loc_6051012
     0001:00000059       __GetExceptDLLinfo
     0001:00000140       sub_6051140
     0001:00000150       loc_6051150
     0001:0000018D       loc_605118D
     0001:0000018F       loc_605118F
*/
    while(!feof(file))
    {
        fgets(str,256,file);
        if(!strncmp(str," 0001:",6))
        {
            char* endptr;
            long offset=strtol(str+6,&endptr,16);//每行偏移6处为相对地址
            if(offset)
            {
                char* ptr=str+21;//每行偏移21处为IDA符号名
                while(*ptr)
                {
                    if(*ptr == '\r' || *ptr == '\n')
                        *ptr='\0';
                    ptr++;
                }
                Quickinsertname(MainModule->codebase+offset,NM_LABEL,str+21);
                Quickinsertname(MainModule->codebase+offset,NM_COMMENT,str+21);
            }
        }
    }
    fclose(file);
    Mergequicknames();
    Addtolist(0,0,"LoadMap: 成功: Map 文件成功导入");
    MessageBox(hOllyWnd,"Map 文件成功导入","LoadMap v0.1",MB_OK|MB_ICONASTERISK);
    Setcpu(0,0,0,0,CPU_ASMFOCUS);
}
 
extc void _export cdecl ODBG_Pluginaction(int origin,int action,void *item)
{
    if(origin != PM_MAIN)
        return;
    if(action == 0)
        LoadMap();
    else if(action == 1)
        MessageBox(hOllyWnd,"LoadMap version 0.1 by lgx/iPB","LoadMap v0.1",MB_OK|MB_ICONASTERISK);
}
```

## 第七章 使用VS2010以上版本的MFC编写OllyDbg插件

&emsp;&emsp;前面已经说过，MFC早期版本的CString实现没有使用ATL，而plugin.h要求编译时加入/J命令，或定义_CHAR_UNSIGNED宏，而ATL要求则相反。因此这个问题在VC6中不存在，而在VS2010及以上版本存在，目前我就测试了这几个版本。如何解决这个矛盾呢？今天我研究出一种方法。  
&emsp;&emsp;#if宏在这里起了决定性作用，如果命令行定义/J则所有工程文件都默认定义_CHAR_UNSIGNED，那么我们退而求其次，只在必要处添加该定义，使用结束后取消定义，这样就不和ATL冲突了！！！下面是该宏用法：

```C++
#if !defined _CHAR_UNSIGNED
#define _CHAR_UNSIGNED
#define _ISCHARTYPEUNSIGNED  1
#else
#define  _ISCHARTYPEUNSIGNED 0
#endif

#if _ISCHARTYPEUNSIGNED == 1
#undef _CHAR_UNSIGNED
#endif
```

&emsp;&emsp;我使用_ISCHARTYPEUNSIGNED 判断是否修改了_CHAR_UNSIGNED，如果原来未定义_CHAR_UNSIGNED，那么就在当前代码块定义该标志，块结束再改回去。如果原来定义过，则不用做任何操作。该代码要和MFC相关宏分开，下面来测试，新建一个MFC的常规DLL，主文件中修改为如下形式：

```C++
// MFCLibrary1.cpp : 定义 DLL 的初始化例程。  
#include "stdafx.h"  
#include "MFCLibrary1.h"  
   
#ifdef _DEBUG  
#define new DEBUG_NEW  
#endif  
   
// CMFCLibrary1App  
   
BEGIN_MESSAGE_MAP(CMFCLibrary1App, CWinApp)  
END_MESSAGE_MAP()  
   
// CMFCLibrary1App 构造  
CMFCLibrary1App::CMFCLibrary1App()  
{  
    // TODO: 在此处添加构造代码，  
    // 将所有重要的初始化放置在 InitInstance 中  
}  
   
// 唯一的一个 CMFCLibrary1App 对象  
CMFCLibrary1App theApp;  

// CMFCLibrary1App 初始化  
BOOL CMFCLibrary1App::InitInstance()  
{  
    CWinApp::InitInstance();  
    return TRUE;  
}  
   
#if !defined _CHAR_UNSIGNED  
#define _CHAR_UNSIGNED  
#define _ISCHARTYPEUNSIGNED  1  
#else  
#define  _ISCHARTYPEUNSIGNED 0  
#endif  
   
#include <Windows.h>  
#include "Plugin.h"  
#include <stdio.h>  
#pragma comment(lib,"ollydbg.lib")  
   
HINSTANCE hInst;  
HWND hOllyWnd;  
t_module* MainModule;  
   
extc int  _export cdecl ODBG_Pluginmenu(int origin,char data[4096],void *item)  
{  
    return 1;  
}  
   
extc int  _export cdecl ODBG_Plugininit(int ollydbgversion,HWND hw, ulong *features)  
{  
    return 0;  
}  
   
extc int  _export cdecl ODBG_Plugindata(char shortname[32])  
{  
    strcat(shortname,"testMFC");  
    return 108;  
}  
   
#if _ISCHARTYPEUNSIGNED == 1  
#undef _CHAR_UNSIGNED  
#endif 


// dllmain.cpp : 定义 DLL 的初始化例程。  
#include "stdafx.h"  
#include <afxwin.h>  
#include <afxdllx.h>  
   
#ifdef _DEBUG  
#define new DEBUG_NEW  
#endif  
   
static AFX_EXTENSION_MODULE MFCLibrary2DLL = { NULL, NULL };  
   
extern "C" int APIENTRY  
DllMain(HINSTANCE hInstance, DWORD dwReason, LPVOID lpReserved)  
{  
    // 如果使用 lpReserved，请将此移除  
    UNREFERENCED_PARAMETER(lpReserved);  
    if (dwReason == DLL_PROCESS_ATTACH)  
    {  
        TRACE0("MFCLibrary2.DLL 正在初始化!\n");   
        // 扩展 DLL 一次性初始化  
        if (!AfxInitExtensionModule(MFCLibrary2DLL, hInstance))  
            return 0;  
        new CDynLinkLibrary(MFCLibrary2DLL);  
    }  
    else if (dwReason == DLL_PROCESS_DETACH)  
    {  
        TRACE0("MFCLibrary2.DLL 正在终止!\n");  
   
        // 在调用析构函数之前终止该库  
        AfxTermExtensionModule(MFCLibrary2DLL);  
    }  
    return 1;   // 确定  
}  
   
#if !defined _CHAR_UNSIGNED  
#define _CHAR_UNSIGNED  
#define _ISCHARTYPEUNSIGNED  1  
#else  
#define  _ISCHARTYPEUNSIGNED 0  
#endif  
   
#include <Windows.h>  
#include "Plugin.h"  
#include <stdio.h>  
#pragma comment(lib,"ollydbg.lib")  
   
HINSTANCE hInst;  
HWND hOllyWnd;  
t_module* MainModule;  
   
extc int  _export cdecl ODBG_Pluginmenu(int origin,char data[4096],void *item)  
{  
    return 1;  
}  
   
extc int  _export cdecl ODBG_Plugininit(int ollydbgversion,HWND hw, ulong *features)  
{  
    return 0;  
}  
   
extc int  _export cdecl ODBG_Plugindata(char shortname[32])  
{  
    strcat(shortname,"testMFC");  
    return 108;  
}  
   
#if _ISCHARTYPEUNSIGNED == 1  
#undef _CHAR_UNSIGNED  
#endif

```