---
layout: post
title: 解决Windows Dll入口函数中Wait系函数死锁的解决
categories: Windows
description: 解决Windows Dll入口函数中Wait系函数死锁的解决
keywords: 
---

## 案例 

&emsp;&emsp;dll中如果用了等待系函数会形成死锁，《Windows核心编程》给出了一个例子

```C++
//***exe.cpp
LoadLibrary(***dll.cpp);

//***dll.cpp
DWORD __stdcall func(LPVOID)
{
	Sleep(2000);
	cout<<"in thread"<<endl;
	return 0;
}
BOOL WINAPI DllMain(HINSTANCE hInstDll,DWORD fdwReason,PVOID fImpLoad)
{
	if(fdwReason == DLL_PROCESS_ATTACH)
	{
		HANDLE hthread=CreateThread(NULL,0,(LPTHREAD_START_ROUTINE)func,NULL,0,NULL);
		if(hthread)
			WaitForSingleObject(hthread);//这里会死锁
	}
	else if(fdwReason == DLL_THREAD_ATTACH)
	{

	}
}
```

## 分析

&emsp;&emsp;这是一个很常见的错误，死锁原因是这样的：在loadlibrary之后，系统加载器执行到dllmain之前，会调用EnterCriticalSection锁住进程的“加载器锁”，而dllmain中的wait函数会共用这个“加载器锁”，再次锁住该锁，因此造成死锁，，，正常的情况是：waitForsingleobject之前就执行完dllmain，之后系统加载器执行LeaveCriticalSection，之后再去wait就不会发生死锁，而现在的情况是，wait在等dllmain执行完，而dllmain在等wait执行完。dllmain中不能用wait系函数也是出自这个原因，如果读了内核代码，会更明白这一块。核心编程的作者说，他能想到的解决办法只有在dllmain中不要出现wait系函数，这种是不良设计思路。事实上MSDN上也说不推荐dllmain中存在耗时操作，所以他们才这么设计的！因为加载dll本身就是很复杂的，很费时间片的工作！！！,核心编程的作者尝试DisableThreadLibraryCall来破解限制，然而没成功。

## 解决

&emsp;&emsp;这里我提出2种方式来破解锁限制，且都可以成功，第一种比较暴力，直接leave掉加载器锁，wait之后再恢复即可，代码如下：

```C++
if(reason == DLL_PROCESS_ATTACH)
{
	cout<<"inside dll!"<<endl;
	PCRITICAL_SECTION loaderlock=NULL;
	_asm
	{
		mov eax,fs:[0x18];
		mov eax,[eax+0x30];
		mov eax,[eax+0xa0];
		mov loaderlock,eax;
	}
	LeaveCriticalSection(loaderlock);
	HANDLE hthread=CreateThread(NULL,0,(LPTHREAD_START_ROUTINE)func,NULL,0,NULL);
	if(hthread)
	{
		cout<<"create succeed!"<<endl;
	}
	WaitForSingleObject(hthread,INFINITE);
	EnterCriticalSection(loaderlock);
}
else if(reason == DLL_THREAD_ATTACH)
{
	cout<<" DLL_THREAD_ATTACH"<<endl;
}

return 0;
```

&emsp;&emsp;这种做法的缺点是，无法保证可重入性，但如果dll足够简单的话，也不妨一试，第二种则采用事件等待的方法，比较规矩：

```C++
void main()
{
	HANDLE hEvent=CreateEventA(NULL,TRUE,FALSE,"Initial");
	HMODULE hmod=LoadLibraryA("C:\\Users\\Administrator\\Documents\\Visual Studio 2010\\Projects\\test1\\Debug\\testdl.dll");
	WaitForSingleObject(hEvent,INFINITE);
	CloseHandle(hEvent);
	cout<<"Success!!"<<endl;
}

DWORD __stdcall func(LPVOID)
{
	cout<<"in thread"<<endl;
	HANDLE hEvent=OpenEventA(EVENT_ALL_ACCESS,TRUE,"Initial");
	if(hEvent)
	{
		SetEvent(hEvent);
		CloseHandle(hEvent);
	}
	return 0;
}

int __stdcall DllMain(int,int reason,int)
{
	if(reason == DLL_PROCESS_ATTACH)
	{
		cout<<"inside dll!"<<endl;
		CreateThread(NULL,0,(LPTHREAD_START_ROUTINE)func,NULL,0,NULL);
	}
	else if(reason == DLL_THREAD_ATTACH)
	{
		cout<<" DLL_THREAD_ATTACH"<<endl;
	}
	return TRUE;
}
```

&emsp;&emsp;为防止滥用，再次强调一下，在dllmain中最好不要放置耗时操作。
