---
layout: post
title: Windows用户态调试学习
categories: Debug
description: windows用户态调试学习
keywords: 
---

# Windows用户态调试学习

## 一 基础知识
&emsp;&emsp;MSDN上的介绍：<https://msdn.microsoft.com/en-us/library/ms679304(v=vs.85).aspx>中的调试事件及调试结构节

### 调试事件：
&emsp;&emsp;WaitForDebugEvent在接收到内核传递的调试信息后会由阻塞态转为激活态，内核会挂起所有线程，直到ContinueDebugEvent执行

|Type                       |Describ        |Struct                   |
|---------------------------|---------------|-------------------------|
|CREATE_PROCESS_DEBUG_EVENT |创建进程事件    |CREATE_PROCESS_DEBUG_INFO|
|CREATE_THREAD_DEBUG_EVENT  |创建线程事件    |CREATE_THREAD_DEBUG_INFO |
|EXCEPTION_DEBUG_EVENT      |发生异常       |EXCEPTION_DEBUG_INFO     |
|EXIT_PROCESS_DEBUG_EVENT   |进程退出       |EXIT_PROCESS_DEBUG_INFO  |
|EXIT_THREAD_DEBUG_EVENT    |线程退出       | EXIT_THREAD_DEBUG_INFO  |
|LOAD_DLL_DEBUG_EVENT       |加载DLL        |LOAD_DLL_DEBUG_INFO      |
|OUTPUT_DEBUG_STRING_EVENT  |输出调试字符串  |OUTPUT_DEBUG_STRING_INFO |
|UNLOAD_DLL_DEBUG_EVENT     |卸载DLL        |UNLOAD_DLL_DEBUG_INFO     |
|RIP_EVENT                  |进程不受系统调试器控制的结束| RIP_INFO      |

### 调试结构：
* CONTEXT：ring3下不能直接读写寄存器，而间接通过设置context结构完成
* DEBUG_EVENT：接受的调试事件结构
* LDT_ENTRY：用于获得描述符表

### 调试函数
* CheckRemoteDebuggerPresent  检查debug port是否存在
* ContinueDebugEvent  继续执行由调试事件打断的线程. 
* DebugActiveProcess  附加调试 
* DebugActiveProcessStop  停止附加调试
* DebugBreak  在当前进程中产生断点
* DebugBreakProcess  在指定进程中产生断点（远程线程注入）
* DebugSetProcessKillOnExit  调试器退出时结束进程
* FatalExit  退出进程并将执行权交给调试器
* FlushInstructionCache  刷新指令缓存（在修改内存指令后使用）
* GetThreadContext  获取线程上下文 
* GetThreadSelectorEntry  获取线程的选择子的描述符表
* IsDebuggerPresent  用调试标志检查用户态调试器是否存在
* OutputDebugString  输出调试信息给调试器
* SetThreadContext  设置线程上下文
* WaitForDebugEvent  等待调试事件

## 二 用户态调试模型及相关代码
&emsp;&emsp;windows用户态调试器是基于事件模式的，由内核做中介的被调试进程和调试器进程的交互，也就是说被调试者和调试器处于不同进程（见《软件调试》），流程如下：

```C++
void main()
 {
         (OpenProcess);
         DebugActiveProcess;
         while(Alive)
         {
                 WaitForDebugEvent;
                 switch(DebugEvent.dwDebugEventCode)
                 {
                         ....
         
                 }
                 ContinueDebugEvent;
         }
 }

&emsp;&emsp;无疑这里最重要的函数就是DebugActiveProcess和ContinueDebugEvent，是调试器最表层的接口，而调试器源码位于ntos/dbgk文件夹下，dbgkobj.c dbgkp.h dbgkport.c dbgkproc.c

```Txt
DebugActiveProcess->NtDebugActiveProcess->DbgkpPostFakeProcessCreateMessages DbgkpSetProcessDebugObject
ContinueDebugEvent->NtDebugContinue->DbgkpWakeTarget->DbgkpFreeDebugEvent
DbgkpPostFakeProcessCreateMessages->DbgkpPostFakeThreadMessages DbgkpPostFakeModuleMessages
```

* DbgkpSendApiMessage 调试消息函数，在某些内核api操作中会将调试事件添加到队列中
* DbgkpSuspendProcess 挂起进程
* DbgkpResumeProcess 恢复线程

&emsp;&emsp;默认下windows只能支持一个调试器，绑定到进程peb的debugport成员中，由于内核代码已经写死了，因此想支持多个调试器同时调试一个进程，只能另寻办法，我的构想是将调试器和内核之间的通信，中间插入一个中介调试器进行协调，同时一个调试器的行为生效则刷新到所有调试器，这样就需要中介调试器hook一些调试api，并完成和这些调试器的通信，而中介调试器直接接收内核发出的调试信息。正在深入研究调试源码并完善设计思路。

```C++
//DbgSsReserved[0]->PDBGSS_THREAD_DATA*
//DbgSsReserved[1]->DebugObjectHandle
BOOL WINAPI ContinueDebugEvent(DWORD dwProcessId, DWORD dwThreadId, DWORD dwContinueStatus)
{
	ClientId.UniqueProcess = (HANDLE)dwProcessId;
	ClientId.UniqueThread = (HANDLE)dwThreadId;
	ZwDebugContinue(NtCurrentTeb()->DbgSsReserved[1], ClientId, dwContinueStatus);
	//移除进程线程句柄
	ThreadData = (PDBGSS_THREAD_DATA*)NtCurrentTeb()->DbgSsReserved;
	ThisData = *ThreadData;
	while(ThisData)
	{
		if ((ThisData->HandleMarked) && ((ThisData->ProcessId == dwProcessId) || (ThisData->ThreadId == dwThreadId)))
		{
			if (ThisData->ThreadHandle) CloseHandle(ThisData->ThreadHandle);
			if (ThisData->ProcessHandle) CloseHandle(ThisData->ProcessHandle);
			*ThreadData = ThisData->Next;
			RtlFreeHeap(RtlGetProcessHeap(), 0, ThisData);
		}
		else
		{
			ThreadData = &ThisData->Next;
		}
		ThisData = *ThreadData;
	}
}

BOOL WINAPI DebugActiveProcess(DWORD dwProcessId)
{
	//创建调试对象
	InitializeObjectAttributes(&ObjectAttributes, NULL, 0, NULL, 0);
	Status = ZwCreateDebugObject(&NtCurrentTeb()->DbgSsReserved[1], DEBUG_OBJECT_ALL_ACCESS, &ObjectAttributes, DBGK_KILL_PROCESS_ON_EXIT);
	//OpenProcess获取进程句柄
	ClientId.UniqueThread = NULL;
	ClientId.UniqueProcess = UlongToHandle(dwProcessId);
	InitializeObjectAttributes(&ObjectAttributes, NULL, 0, NULL, NULL);
	Status = NtOpenProcess(&Handle, PROCESS_CREATE_THREAD | PROCESS_VM_OPERATION | PROCESS_VM_WRITE | PROCESS_VM_READ 
		| PROCESS_SUSPEND_RESUME | PROCESS_QUERY_INFORMATION, &ObjectAttributes, &ClientId);
	Status = NtDebugActiveProcess(Process, NtCurrentTeb()->DbgSsReserved[1]);
	//注入并设置int 3
	NtClose(Handle);
}

BOOL WINAPI DebugActiveProcessStop(DWORD dwProcessId)
{
	//OpenProcess获取进程句柄
	ClientId.UniqueThread = NULL;
	ClientId.UniqueProcess = UlongToHandle(dwProcessId);
	InitializeObjectAttributes(&ObjectAttributes, NULL, 0, NULL, NULL);
	Status = NtOpenProcess(&Handle, PROCESS_CREATE_THREAD | PROCESS_VM_OPERATION | PROCESS_VM_WRITE | PROCESS_VM_READ 
		| PROCESS_SUSPEND_RESUME | PROCESS_QUERY_INFORMATION, &ObjectAttributes, &ClientId);
	//关闭所有进程句柄
	ThreadData = (PDBGSS_THREAD_DATA*)NtCurrentTeb()->DbgSsReserved;
	ThisData = *ThreadData;
	while(ThisData)
	{
		if (ThisData->ProcessId == dwProcessId)
		{
			if (ThisData->ThreadHandle) CloseHandle(ThisData->ThreadHandle);
			if (ThisData->ProcessHandle) CloseHandle(ThisData->ProcessHandle);
			*ThreadData = ThisData->Next;
			RtlFreeHeap(RtlGetProcessHeap(), 0, ThisData);
		}
		else
		{
			ThreadData = &ThisData->Next;
		}
		ThisData = *ThreadData;
	}
	Status = NtRemoveProcessDebug(Process, NtCurrentTeb()->DbgSsReserved[1]);
	NtClose(Handle);
}

BOOL WINAPI DebugSetProcessKillOnExit(BOOL KillOnExit)
{
	STATE = KillOnExit != 0;
	Handle = NtCurrentTeb()->DbgSsReserved[1];
	Status = NtSetInformationDebugObject(Handle, DebugObjectKillProcessOnExitInformation, &State, sizeof(State), NULL);
}

BOOL WINAPI WaitForDebugEvent(LPDEBUG_EVENT lpDebugEvent, DWORD dwMilliseconds)
{
	do
	{
		Status = NtWaitForDebugEvent(NtCurrentTeb()->DbgSsReserved[1], TRUE, TimeOut, &WaitStateChange);
	} while ((Status == STATUS_ALERTED) || (Status == STATUS_USER_APC));

	lpDebugEvent->dwProcessId = (DWORD)WaitStateChange->AppClientId.UniqueProcess;
	lpDebugEvent->dwThreadId = (DWORD)WaitStateChange->AppClientId.UniqueThread;

	switch (WaitStateChange->NewState)
	{
	case DbgCreateThreadStateChange:
		DebugEvent->dwDebugEventCode = CREATE_THREAD_DEBUG_EVENT;
		DebugEvent->u.CreateThread.hThread = WaitStateChange->StateInfo.CreateThread.HandleToThread;
		DebugEvent->u.CreateThread.lpStartAddress = WaitStateChange->StateInfo.CreateThread.NewThread.StartAddress;
		Status = NtQueryInformationThread(WaitStateChange->StateInfo.CreateThread.HandleToThread, ThreadBasicInformation, &ThreadBasicInfo, sizeof(ThreadBasicInfo), NULL);
		DebugEvent->u.CreateThread.lpThreadLocalBase = ThreadBasicInfo.TebBaseAddress;
		break;

	case DbgCreateProcessStateChange:
		DebugEvent->dwDebugEventCode = CREATE_PROCESS_DEBUG_EVENT;
		DebugEvent->u.CreateProcessInfo.hProcess = WaitStateChange->StateInfo.CreateProcessInfo.HandleToProcess;
		DebugEvent->u.CreateProcessInfo.hThread = WaitStateChange->StateInfo.CreateProcessInfo.HandleToThread;
		DebugEvent->u.CreateProcessInfo.hFile = WaitStateChange->StateInfo.CreateProcessInfo.NewProcess.FileHandle;
		DebugEvent->u.CreateProcessInfo.lpBaseOfImage = WaitStateChange->StateInfo.CreateProcessInfo.NewProcess.BaseOfImage;
		DebugEvent->u.CreateProcessInfo.dwDebugInfoFileOffset = WaitStateChange->StateInfo.CreateProcessInfo.NewProcess.DebugInfoFileOffset;
		DebugEvent->u.CreateProcessInfo.nDebugInfoSize = WaitStateChange->StateInfo.CreateProcessInfo.NewProcess.DebugInfoSize;
		DebugEvent->u.CreateProcessInfo.lpStartAddress = WaitStateChange->StateInfo.CreateProcessInfo.NewProcess.InitialThread.StartAddress;
		Status = NtQueryInformationThread(WaitStateChange->StateInfo.CreateProcessInfo.HandleToThread, ThreadBasicInformation,
			&ThreadBasicInfo, sizeof(ThreadBasicInfo), NULL);
		DebugEvent->u.CreateProcessInfo.lpThreadLocalBase = ThreadBasicInfo.TebBaseAddress;
		DebugEvent->u.CreateProcessInfo.lpImageName = NULL;
		DebugEvent->u.CreateProcessInfo.fUnicode = TRUE;
		break;

	case DbgExitThreadStateChange:
		DebugEvent->dwDebugEventCode = EXIT_THREAD_DEBUG_EVENT;
		DebugEvent->u.ExitThread.dwExitCode = WaitStateChange->StateInfo.ExitThread.ExitStatus;
		break;

	case DbgExitProcessStateChange:
		DebugEvent->dwDebugEventCode = EXIT_PROCESS_DEBUG_EVENT;
		DebugEvent->u.ExitProcess.dwExitCode = WaitStateChange->StateInfo.ExitProcess.ExitStatus;
		break;

	case DbgExceptionStateChange:
	case DbgBreakpointStateChange:
	case DbgSingleStepStateChange:
		if (WaitStateChange->StateInfo.Exception.ExceptionRecord.ExceptionCode == DBG_PRINTEXCEPTION_C)
		{
			DebugEvent->dwDebugEventCode = OUTPUT_DEBUG_STRING_EVENT;
			DebugEvent->u.DebugString.lpDebugStringData = (PVOID)WaitStateChange->StateInfo.Exception.ExceptionRecord.ExceptionInformation[1];
			DebugEvent->u.DebugString.nDebugStringLength = WaitStateChange->StateInfo.Exception.ExceptionRecord.ExceptionInformation[0];
			DebugEvent->u.DebugString.fUnicode = FALSE;
		}
		else if (WaitStateChange->StateInfo.Exception.ExceptionRecord.ExceptionCode == DBG_RIPEXCEPTION)
		{
			DebugEvent->dwDebugEventCode = RIP_EVENT;
			DebugEvent->u.RipInfo.dwType = WaitStateChange->StateInfo.Exception.ExceptionRecord.ExceptionInformation[1];
			DebugEvent->u.RipInfo.dwError = WaitStateChange->StateInfo.Exception.ExceptionRecord.ExceptionInformation[0];
		}
		else
		{
			DebugEvent->dwDebugEventCode = EXCEPTION_DEBUG_EVENT;
			DebugEvent->u.Exception.ExceptionRecord = WaitStateChange->StateInfo.Exception.ExceptionRecord;
			DebugEvent->u.Exception.dwFirstChance = WaitStateChange->StateInfo.Exception.FirstChance;
		}
		break;

	case DbgLoadDllStateChange:
		DebugEvent->dwDebugEventCode = LOAD_DLL_DEBUG_EVENT;
		DebugEvent->u.LoadDll.lpBaseOfDll = WaitStateChange->StateInfo.LoadDll.BaseOfDll;
		DebugEvent->u.LoadDll.hFile = WaitStateChange->StateInfo.LoadDll.FileHandle;
		DebugEvent->u.LoadDll.dwDebugInfoFileOffset = WaitStateChange->StateInfo.LoadDll.DebugInfoFileOffset;
		DebugEvent->u.LoadDll.nDebugInfoSize = WaitStateChange->StateInfo.LoadDll.DebugInfoSize;
		InitializeObjectAttributes(&ObjectAttributes, NULL, 0, NULL, NULL);
		Status = NtOpenThread(&ThreadHandle, THREAD_QUERY_INFORMATION, &ObjectAttributes, &WaitStateChange->AppClientId);
		Status = NtQueryInformationThread(ThreadHandle, ThreadBasicInformation, &ThreadBasicInfo, sizeof(ThreadBasicInfo), NULL);
		NtClose(ThreadHandle);
		DebugEvent->u.LoadDll.lpImageName = ((PTEB)ThreadBasicInfo.TebBaseAddress)->NtTib.ArbitraryUserPointer;
		DebugEvent->u.LoadDll.fUnicode = TRUE;
		break;

	case DbgUnloadDllStateChange:
		DebugEvent->dwDebugEventCode = UNLOAD_DLL_DEBUG_EVENT;
		DebugEvent->u.UnloadDll.lpBaseOfDll = WaitStateChange->StateInfo.UnloadDll.BaseAddress;
		break;

	default: 
		return STATUS_UNSUCCESSFUL;
	}

	switch (lpDebugEvent->dwDebugEventCode)
	{
	case CREATE_THREAD_DEBUG_EVENT:
		SaveThreadHandle(lpDebugEvent->dwProcessId, lpDebugEvent->dwThreadId, lpDebugEvent->u.CreateThread.hThread);
		break;
	case CREATE_PROCESS_DEBUG_EVENT:
		SaveProcessHandle(lpDebugEvent->dwProcessId, lpDebugEvent->u.CreateProcessInfo.hProcess);
		SaveThreadHandle(lpDebugEvent->dwProcessId, lpDebugEvent->dwThreadId, lpDebugEvent->u.CreateProcessInfo.hThread);
		break;
	case EXIT_PROCESS_DEBUG_EVENT:
		MarkProcessHandle(lpDebugEvent->dwProcessId);
	case EXIT_THREAD_DEBUG_EVENT:
		MarkThreadHandle(lpDebugEvent->dwThreadId);
		break;
	case EXCEPTION_DEBUG_EVENT:
	case LOAD_DLL_DEBUG_EVENT:
	case UNLOAD_DLL_DEBUG_EVENT:
	case OUTPUT_DEBUG_STRING_EVENT:
	case RIP_EVENT:
		break;
	default:
		return FALSE;
	}
}

#define DbgSsSetThreadData(d) NtCurrentTeb()->DbgSsReserved[0] = d
#define DbgSsGetThreadData() ((PDBGSS_THREAD_DATA)NtCurrentTeb()->DbgSsReserved[0])

VOID WINAPI SaveThreadHandle(DWORD dwProcessId, DWORD dwThreadId, HANDLE hThread)
{
	PDBGSS_THREAD_DATA ThreadData;
	ThreadData = RtlAllocateHeap(RtlGetProcessHeap(), 0, sizeof(DBGSS_THREAD_DATA));
	ThreadData->ThreadHandle = hThread;
	ThreadData->ProcessId = dwProcessId;
	ThreadData->ThreadId = dwThreadId;
	ThreadData->ProcessHandle = NULL;
	ThreadData->HandleMarked = FALSE;
	ThreadData->Next = DbgSsGetThreadData();
	DbgSsSetThreadData(ThreadData);
}

VOID WINAPI SaveProcessHandle(DWORD dwProcessId, HANDLE hProcess)
{
	PDBGSS_THREAD_DATA ThreadData;
	ThreadData = RtlAllocateHeap(RtlGetProcessHeap(), 0, sizeof(DBGSS_THREAD_DATA));
	ThreadData->ProcessHandle = hProcess;
	ThreadData->ProcessId = dwProcessId;
	ThreadData->ThreadId = 0;
	ThreadData->ThreadHandle = NULL;
	ThreadData->HandleMarked = FALSE;
	ThreadData->Next = DbgSsGetThreadData();
	DbgSsSetThreadData(ThreadData);
}

VOID WINAPI MarkProcessHandle(DWORD dwProcessId)
{
	PDBGSS_THREAD_DATA ThreadData;
	for (ThreadData = DbgSsGetThreadData(); ThreadData; ThreadData = ThreadData->Next)
	{
		if ((ThreadData->ProcessId == dwProcessId) && !(ThreadData->ThreadId))
		{
			ThreadData->HandleMarked = TRUE;
			break;
		}
	}
}

VOID WINAPI MarkThreadHandle(DWORD dwThreadId)
{
	PDBGSS_THREAD_DATA ThreadData;
	for (ThreadData = DbgSsGetThreadData(); ThreadData; ThreadData = ThreadData->Next)
	{
		if (ThreadData->ThreadId == dwThreadId)
		{
			ThreadData->HandleMarked = TRUE;
			break;
		}
	}
}
```

可见作为内核接口的api有：
* ZwDebugContinue
* ZwCreateDebugObject
* NtDebugActiveProcess
* NtRemoveProcessDebug
* NtSetInformationDebugObject
* NtWaitForDebugEvent

