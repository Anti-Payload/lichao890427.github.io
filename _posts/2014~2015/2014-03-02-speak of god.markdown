---
layout: post
title: 和驱动大神的深入对话
categories: WindowsDriver
description: 和驱动大神的深入对话
keywords: 
---
Q stands for Question;A stands for Answer  
A:如果想拦截运行程序的API，NT5系统拦截NtCreateProcessEx，NT6系统拦截NtCreateUserProcess即可  
Q:那已经是驱动级的函数了  
A:不是。  
Q:显然是  
A:在RING3拦截NTDLL里的这两个函数即可  
Q:你告诉我Nt开头的不是内核函数么  
A:他是应用层的函数？？  
Q:是的。  
A:是应用层的函数。  
Q:恩，还真是呢！  
A:也是内核层的函数。NTDLL里有这两个函数 SSDT里也有这两个函数拦截东西为了干脆，直接拦截NTDLL里的函数即可。  
Q:对啊，这个伟大的工作应该由你完成的。。。这个对你很easy  
A:比如你要拦截网络SEND RECV，直接拦截NTDLL的NtDeviceIoControlFile即可。。。   
Q:我现在还没见过任意一款可以拦截任意api的工具出现  
A:话说这些东西网上都有好吧。。。  
Q:恩，你是怎么拦截的呢  
A:32位系统下，直接hook ntdll!kifastsystemcall  
Q:即可拦截一个程序所有的API调用  
A:64位系统下，因为NTDLL里，直接通过SYSCALL指令进入内核，所以要拦截的话，只能通过VT-X技术拦截了  
Q:你怎么hook，改写内核函数？  
A:这个不是内核函数。。。  
Q:你是怎么hook的，都到ntdll了，你居然跟我说不是内核函数  
A:是R3的函数。关于HOOK API,有两个比较好的模板：1.MHOOK  2.MINIHOOKENGINE，要不我发个例子到群共享如何？  
Q:恩，可以啊  
A:RING3的，我找找啊，找到了一个hook NtOpenFile的例子  
Q:这个改改就可  
A:呵呵，你又得跟我说ntopenfile是内核函数了，目测                                                                                                   
Q:必须得有一个进程吗？这个                                                                                                                         
A:Usage is :inject <process> <full dll path>                                                                                                       
Q:不能一次拦截系统所有的？                                                                                                                         
A:哎。我都说了。NtOpenFile，在NTDLL和SSDT里都有。NTDLL里的不是内核函数，SSDT里的才是。                                                             
而且R3的API HOOK，只能针对一个进程，R0的API HOOK才是针对所有进程有效。                                                                             
Q:如果你直接改了内核里函数的代码，让他跳转到你的代码，那应该所有进程都给拦截到了吧？                                                               
A:你这个貌似不是这么实现的                                                                                                                         
Q:那就需要驱动了。                                                                                                                                 
A:R3是无权改内核空间代码的。XP之后系统。                                                                                                           
Q:不是啊，我是说改dll的                                                                                                                            
A:改应用层的                                                                                                                                       
Q:不行的。只能拦截一个                                                                                                                             
A:比如CreateFile这种，我不改NtCreateFile                                                                                                           
Q:这是为啥                                                                                                                                         
A:代码不是共用的吗？他们                                                                                                                           
Q:因为你只是修改了一个进程的空间啊。不影响别的进程的。比如说，A、B两个进程都有OPENPROCESS，你修改了A进程的OPENPROCESS,不影响B进程的OPENPROCESS     
A:但是他们不是都用一个dll的东西吗，只不过这个dll的代码被映射到2个空间？                                                                            
Q:我觉得有可能会影响到……的吧？DLL是共用的吧？改了一个代码为啥改不掉另外一个？                                                                      
A:WINDOWS有一个机制，叫做COPY-ON-WRITE                                                                                                             
Q:也就是说还是copy了。。。                                                                                                                         
A:你写入的时候，对应的物理地址会变化的。                                                                                                           
Q:那到底写到哪里去了？                                                                                                                             
A:话说，这些涉及到WINDOWS内存管理的知识，一下子说不清楚。不忽悠地说，你可以看一下WINDOWS INTERNALS                                                 
Q:我就想简单地知道我这个代码究竟改到哪里去了，既然他没有改到“公用的”代码里                                                                         
A:但可以肯定的一个“事实”是：你修改了A进程的XXX函数，不会影响到B进程的XXX函数                                                                       
@Q 这个A是真的能通过cr3来实现注入的，也就是不使用OpenProcess等API                                                                                  
Q:现在你的说法就是dll在加载的时候，不是共享代码那么简单，，，事实上没有共享代码?                                                                   
A:关于HOOK API，有两个成熟稳定的实现：                                                                                                             
http://codefromthe70s.org/mhook.aspx
http://www.codeproject.com/Artic ... 64-Mini-Hook-Engine
Q:其实还是copy了dll的代码到各自空间？
我已经知道了，当你修改DLL所在的内存区间的时候，Win自动复制了一份DLL公用，你改的部分只对你的EXE有效。DLL刚载入，还没有修改的时候，是所有进程公用的，但是如果有个进程改了DLL的内存，哎，，那这个dll还说可以共享的，还不占内存，不是扯淡么   
A:这个没啥，用WINDBG查看一下就行了~                                                                                      
Q:Win就专门给这个DLL复制，也就是说，你不改DLL，Win就不复制                                                               
A:所有的书的dll都这么介绍的，什么内存共享，尼玛实际上不是那么实现的。。。。。其实是书上没写清楚，或者是你没有理解好。    
A:DLL只在被你修改的时候，才不共享内存，而且也只是修改的那一块                                                            
Q:共享和不共享差异太大了吧                                                                                               
A:你若对此有疑问 搜索一下COPY-ON-WRITEj机制即可。                                                                        
Q:那你不能这么说，运行的时候谁去修改dll。。。                                                                            
A:修改系统文件本来就是一件很不可能的事，除非你有特殊需求                                                                 
http://blog.csdn.net/freexploit/article/details/275360
http://www.cnblogs.com/super119/archive/2011/04/10/2011404.html
这两篇文章有一定的参考价值  
A:修改DLL的时候会引起中断，Win对这个中断的处理方式，就是帮你复制一份dll中有数据，代码一般是不改的，改数据的话每个进程就用不同的数据页了，只是修改的页不是整个DLL，我的猜想……    
Q:也就说他干的就是破坏操作系统的事，的所有的事都是绕过操作系统，用内核级函数执行    
Q:那他到底改到哪里去了。。。   
A:没这么神  不过是对WINDOWS OS的了解稍微多这么一丁点而已  
Q:而且改了他得执行吧，他究竟在哪里执行的？  
A:恩，也就说数据段是共享的，代码段另分配？？？  
Q:这和一般程序执行原理不大相符吧    
A:你们能不能聊一些我们看不懂的话题   
每个进程有自己的线性地址映射，dll的话一开始每个进程映射的页是一样的，随着数据的改变，映射不一样了，这就是写时复制  
Q:恩，我不管他怎么从虚拟内存映射到物理内存，我只看虚拟内存部分。。   
为啥代码段另分配    
A:@Q 您用WINDBG做个实验即可。  
首先调试运行2个计算器（A、B）  
* 1.分别用UF ShellAboutW，查看反汇编代码，可以发现两者一样
* 2.用EB ShellAboutW 0修改其中一个进程
* 3.再次用UF ShellAboutW，查看反汇编代码，可以发现两者不一样了。  
这个就是写时复制   

A:没有另分配，只是数据在一堆，代码在一堆   
A:另外，内存有分虚拟内存和物理内存之分，一个物理内存可以对应多个虚拟内存，但反之不行。物理内存有一个属性，名为MMPTE_HARDWARE：  
```C++
typedef struct _MMPTE_HARDWARE
{
	ULONGLONG Valid : 1;
	ULONGLONG Write : 1;        //[这里才是跟Copy-On-WRITE有关的位置]
	ULONGLONG Owner : 1;
	ULONGLONG WriteThrough : 1;
	ULONGLONG CacheDisable : 1;
	ULONGLONG Accessed : 1;
	ULONGLONG Dirty : 1;
	ULONGLONG LargePage : 1;
	ULONGLONG Global : 1;
	ULONGLONG CopyOnWrite : 1; // software field
	ULONGLONG Prototype : 1;   // software field
	ULONGLONG reserved0 : 1;  // software field
	ULONGLONG PageFrameNumber : 28;
	ULONG64 reserved1 : 24 - (_HARDWARE_PTE_WORKING_SET_BITS+1);
	ULONGLONG SoftwareWsIndex :
	_HARDWARE_PTE_WORKING_SET_BITS;
	ULONG64 NoExecute : 1;
} MMPTE_HARDWARE, *PMMPTE_HARDWARE;
```

Q:一个物理内存可以对应多个虚拟内存，但反之不行。但是这样就更诡异了。。。    
Q:你说dll每次加载的时候，其实代码部分还是copy到一个新的物理内存去？数据是都在一个物理内存，然后这样映射到虚拟内存的？这个意思？A:如果想达到你说的效果，修改一处统统生效，则要在没有修改之前，把API地址（虚拟内存）对应的物理内存的MMPTE_HARDWARE.Write设置为1  
Q:不是，是你修改的时候，才复制。  
A:不修改就不复制  
Q:哦  
copy的只是页表指针
然后修改了以后，就弄到一个新的物理地址处，还是映射到原来的虚拟内存了？
所以你改的其实是一个新的地方  
A:一个新的物理地址出，WINDOWS的内存管理极为复杂，我花了一个月时间，也就弄懂一点皮毛。。。  
Q:恩，这样倒是说得通，dll也是内存共享的。。没啥冲突  
A:这么说吧，当你修改了某个进程的API指令时，你看似地址一样，假设都是77665544，但实际上两个进程里，77665544对应的物理地址已经不同了  
Q:恩，我知道，只有关了writeoncopy他才会在原来的地址上改
A:另外，虚拟地址转换为物理地址也非常麻烦，WIN32系统是3层表，WIN64系统是4层表       对  
Q:然后你的工具就是自己实现转换么？你现有的这些内核驱动知识，是不是看了很多论坛的结果？比如看雪这种？  
A:内核里有函数，名为MmGetPhysicalAddress      另外我不上看雪。。。这些问题我一般上kernelmode.info，我问问题，一般上上kernelmode.info，老大都是俄罗斯著名黑客
其中一个人，是PATCHGUARD团队的成员。。。  http://forum.sysinternals.com/      这个也还可以
Q:sysinternals这个我知道哦，这个人写了一堆工具，最后被微软要走了  
A:老实说，真心学东西的话，华人多的地方别去，分享少，吵架多。  
Q:不过大部分也是应用层的。。  
A:个人感觉，WINDOWS内核方面，牛人最多，而且乐于助人的论坛，非kernelmode.info莫属。。。  
Q:什么regmon procmon filemon以前经常用  
A:看雪基本当成资料下载站。  
Q:我还想问你现在的知道这些花了多长时间？  
A:我也不是全部时间搞这个的。编程只是我生活的一部分。。。断断续续弄了5年时间吧。每天2~5小时来算  
Q:恩，那你的工作是？  
A:公务员~~~可惜没啥油水啊     一个月工资4000块钱  
Q:以前是有的  
Q:对，你能不能破解qq的聊天记录  
A:我只懂点WINDOWS编程，破解和逆向略懂皮毛而已。。。这个我老实说：做不到。  
Q:但是我觉得驱动和内核，尤其是win的，代码比较少，如果不去逆向别人的东西，怎么知道咋做的呢，我一直以为搞这个是靠逆向的  
A:这个貌似我运气比较好吧  当年遇到贵人带我入门  
Q:我咋没遇到  
A:没啥，WINDOWS内核这块其实没啥意思，还是搞通用编程好。不用考虑系统，话说，我大学的毕业设计是DBMS,毕业后就没搞了。因为没法盈利~  
Q:如果真想写一个底层的东西，比如病毒木马或者杀毒软件，是不是只要把所有应用api和内核api全实现一遍就无敌了？是不？  
A:有人这么做。这玩意叫作RELOAD KERNEL  
Q:做出来就是超大型软件  
A:你这么说其实倒也没错，不过难度太大，一入内核神似海 ，里面见不得光的东西太多了，深  
