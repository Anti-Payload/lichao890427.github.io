---
layout: post
title: 研究spy++如何获取窗口的WndProc回调地址
categories: Reverse_Engineering
description: 研究spy++如何获取窗口的WndProc回调地址
keywords: 
---

## 背景

&emsp;&emsp;该题目源于网上一段提问，提问者问：每个窗口都有其回调函数WndProc用于处理消息，然而如果获取其他进程的窗口信息，通过GetWindowLong仅能获取到除了WndProc以外的参数，唯独WndProc获取不了，然而为何Spy++可以？是通过HWND对应的内核结构获取到的吗？（内核驱动我不懂，懂得大哥说说）  

## 源码 

&emsp;&emsp;经过实验我发现确实不能用GetWindowLong获取到该参数GWL_WNDPROC，于是我决定研究一下Spy++，弄了一天终于研究出来了。分析表明Spy++采取了Windows自带Hook功能获取一些特殊窗口参数，关键代码在spyxxhk.dll的SpyxxGetMsgProc函数中。试验方法是：随便用spy++查找一个窗口，右键查看窗口属性，看常规选项卡中“窗口进程”的结果，例如00417912，之后用WinHex打开Spyxx.exe进程的SPYXXHK.DLL模块，搜索该数据，发现在6D325C44处，使用IDA附加该进程，查看该处地址为一变量，查看引用可知SpyxxGetMsgProc中对其进行了修改。接下来看Spyxx.exe对该函数的调用，可以分析出如下代码：

```C++
HMODULE spyhkmod;  
HHOOK getmsghook;  
HHOOK callwndprochook;  
HHOOK callwndprocrethook;  
extern HHOOK _ghhkMsgHook;//dll导出变量，对应getmsghook  
extern HHOOK _ghhkCallHook;//dll导出变量，对应callwndprochook  
extern HHOOK _ghhkRetHook;//dll导出变量，对应callwndprocrethook  
   
void SetMsgHook(BOOL Enable)  
{  
    if(Enable)//hook  
    {  
        if(!spyhkmod)  
        {  
            spyhkmod=GetModuleHandle(_T("SpyxxHk"));  
            if(!spyhkmod)  
                return;  
        }  
        if(!getmsghook)  
        {  
            getmsghook=SetWindowsHookEx(WH_GETMESSAGE,SpyxxGetMsgProc,spyhkmod,0);  
            if(!getmsghook)  
            {  
                CString text,caption;  
                text.LoadString(113);//Cannot set the WH_GETMESSAGE hook.  Message logging is inoperable.  
                caption.LoadString(1);//Microsoft Spy++  
                MessageBox(NULL,text,caption,MB_OK|MB_ICONEXCLAMATION|MB_SYSTEMMODAL);  
                return;  
            }  
            _ghhkMsgHook=getmsghook;  
        }  
        if(!callwndprochook)  
        {  
            callwndprochook=SetWindowsHookEx(WH_CALLWNDPROC,SpyxxCallWndProc,spyhkmod,0);  
            if(!callwndprochook)  
            {  
                CString text,caption;  
                text.LoadString(114);//Cannot set the WH_CALLWNDPROC hook.  Message logging is inoperable.  
                caption.LoadString(1);//Microsoft Spy++  
                MessageBox(NULL,text,caption,MB_OK|MB_ICONEXCLAMATION|MB_SYSTEMMODAL);  
                UnhookWindowsHookEx(getmsghook);  
                return;  
            }  
            _ghhkCallHook=callwndprochook;  
        }  
        if(!callwndprocrethook)  
        {  
            callwndprocrethook=SetWindowsHookEx(WH_CALLWNDPROCRET,SpyxxCallWndRetProc,spyhkmod,0);  
            if(!callwndprocrethook)  
            {  
                CString text,caption;  
                text.LoadString(115);//,Cannot set the WH_CALLWNDPROCRET hook.  Message logging is inoperable.  
                caption.LoadString(1);//Microsoft Spy++  
                MessageBox(NULL,text,caption,MB_OK|MB_ICONEXCLAMATION|MB_SYSTEMMODAL);  
                UnhookWindowsHookEx(getmsghook);  
                UnhookWindowsHookEx(callwndprochook);  
                return;  
            }  
            _ghhkRetHook=callwndprocrethook;  
        }  
    }  
    else//unhook  
    {  
        if(getmsghook)  
        {  
            UnhookWindowsHookEx(getmsghook);  
            getmsghook=NULL;  
        }  
        if(callwndprochook)  
        {  
            UnhookWindowsHookEx(callwndprochook);  
            callwndprochook=NULL;  
        }  
        //还有一个没有unhook呢？bug？  
    }  
}  
   
HWND hWnd;  
HANDLE hWriterMutex,hAccessMutex,hReadEvent,hWrittenEvent,hOtherAccessMutex,hOtherDataEvent;  
void HookMain(void* param)  
{  
    hWriterMutex = CreateMutexW(0, 0, L"Local\\Spy++ Writer Mutex");  
    hAccessMutex = CreateMutexW(0, 0, L"Local\\Spy++ Access Mutex");  
    hReadEvent = CreateEventW(0, 0, 1, L"Local\\Spy++ Read Event");  
    hWrittenEvent = CreateEventW(0, 0, 0, L"Local\\Spy++ Written Event");  
    hOtherAccessMutex = CreateMutexW(0, 0, L"Local\\Spy++ Other Process Access Mutex");  
    hOtherDataEvent = CreateEventW(0, 0, 0, L"Local\\Spy++ Other Process Data Event");  
    PSECURITY_DESCRIPTOR sd=NULL;  
    if(ConvertStringSecurityDescriptorToSecurityDescriptor(  
        _T("D:(A;;GA;;;WD)(A;;GA;;;SY)(A;;GA;;;BA)(A;;GA;;;AN)(A;;GA;;;RC)(A;;GA;;;S-1-15-2-1)"),  
        SDDL_REVISION_1,&sd,NULL))  
    {  
        PACL pDacl;  
        BOOL bDaclPresent;  
        BOOL bDaclDefaulted;  
        if(GetSecurityDescriptorDacl(sd,&bDaclPresent,&pDacl,&bDaclDefaulted))  
        {  
            SetSecurityInfo(hWriterMutex,SE_KERNEL_OBJECT,DACL_SECURITY_INFORMATION,NULL,NULL,pDacl,NULL);  
            SetSecurityInfo(hAccessMutex,SE_KERNEL_OBJECT,DACL_SECURITY_INFORMATION,NULL,NULL,pDacl,NULL);  
            SetSecurityInfo(hReadEvent,SE_KERNEL_OBJECT,DACL_SECURITY_INFORMATION,NULL,NULL,pDacl,NULL);  
            SetSecurityInfo(hWrittenEvent,SE_KERNEL_OBJECT,DACL_SECURITY_INFORMATION,NULL,NULL,pDacl,NULL);  
            SetSecurityInfo(hOtherAccessMutex,SE_KERNEL_OBJECT,DACL_SECURITY_INFORMATION,NULL,NULL,pDacl,NULL);  
            SetSecurityInfo(hOtherDataEvent,SE_KERNEL_OBJECT,DACL_SECURITY_INFORMATION,NULL,NULL,pDacl,NULL);  
        }  
        LocalFree(sd);  
    }  
    if(ConvertStringSecurityDescriptorToSecurityDescriptor(_T("S:(ML;;NW;;;LW)"),SDDL_REVISION_1,&sd,NULL))  
    {  
        PACL pSacl;  
        BOOL bDaclPresent;  
        BOOL bDaclDefaulted;  
        if(GetSecurityDescriptorDacl(sd,&bDaclPresent,&pSacl,&bDaclDefaulted))  
        {  
            SetSecurityInfo(hWriterMutex,SE_KERNEL_OBJECT,LABEL_SECURITY_INFORMATION,NULL,NULL,pSacl,NULL);  
            SetSecurityInfo(hAccessMutex,SE_KERNEL_OBJECT,LABEL_SECURITY_INFORMATION,NULL,NULL,pSacl,NULL);  
            SetSecurityInfo(hReadEvent,SE_KERNEL_OBJECT,LABEL_SECURITY_INFORMATION,NULL,NULL,pSacl,NULL);  
            SetSecurityInfo(hWrittenEvent,SE_KERNEL_OBJECT,LABEL_SECURITY_INFORMATION,NULL,NULL,pSacl,NULL);  
            SetSecurityInfo(hOtherAccessMutex,SE_KERNEL_OBJECT,LABEL_SECURITY_INFORMATION,NULL,NULL,pSacl,NULL);  
            SetSecurityInfo(hOtherDataEvent,SE_KERNEL_OBJECT,LABEL_SECURITY_INFORMATION,NULL,NULL,pSacl,NULL);  
        }  
        LocalFree(sd);  
    }  
    //.........  
    SetMsgHook(TRUE);  
    //........  
}  
void WINAPI CreateHookThread()  
{  
    //.........  
    _beginthread(HookMain,0x10000,hWnd);  
    //.........  
}  
int CSpyApp::InitInstance()  
{  
    //.........  
    CreateHookThread();  
    //.........  
}  
BOOL gfHookEnabled,gfHookDisabled;  
DWORD _gpidSpyxx;  
LRESULT WINAPI SpyxxCallWndProc(int nCode, WPARAM wParam, LPARAM lParam)  
{  
    LRESULT ret=CallNextHookEx(_ghhkCallHook,nCode,wParam,lParam);  
    CWPSTRUCT* cwp=(CWPSTRUCT*)lParam;  
    if(gfHookEnabled && !gfHookDisabled && nCode == HC_ACTION && cwp && cwp->hwnd)  
    {      
         DWORD procid;  
        DWORD id=GetWindowThreadProcessId(cwp->hwnd,&procid);  
        if(procid != _gpidSpyxx && id != _gpidSpyxx && GetCurrentThreadId() != _gpidSpyxx)  
            func(1,0,cwp->hwnd,cwp->message,cwp->wParam,cwp->lParam,0,0,0);  
    }  
    return ret;  
}  
LRESULT WINAPI SpyxxCallWndRetProc(int nCode, WPARAM wParam, LPARAM lParam)  
{  
    LRESULT ret=CallNextHookEx(_ghhkRetHook,nCode,wParam,lParam);  
    CWPRETSTRUCT* cwp=(CWPRETSTRUCT*)lParam;  
    if(gfHookEnabled && !gfHookDisabled && nCode == HC_ACTION && cwp && cwp->hwnd)  
    {      
         DWORD procid;  
        DWORD id=GetWindowThreadProcessId(cwp->hwnd,&procid);  
        if(procid != _gpidSpyxx && id != _gpidSpyxx && GetCurrentThreadId() != _gpidSpyxx)  
            func(2,cwp->lResult,cwp->hwnd,cwp->message,cwp->wParam,cwp->lParam,0,0,0);  
    }  
    return ret;  
}  
UINT gmsgOtherProcessData;  
HWND gopd;//目标进程窗口句柄，exe在OnIdle例程中修改该处  
DWORD goproc;//目标窗口回调函数地址  
WNDCLASS WndClass;  
LRESULT WINAPI SpyxxGetMsgProc(int nCode, WPARAM wParam, LPARAM lParam)  
{  
    MSG* msg=(MSG*)lParam;  
    if(nCode == HC_ACTION && (wParam&PM_REMOVE) && msg && msg->hwnd)  
    {  
        if(msg->message == gmsgOtherProcessData)  
        {  
            if(msg->hwnd == gopd && hAccessMutex && !WaitForSingleObject(hAccessMutex,1000))  
            {  
                TCHAR ClassName[128];  
                GetClassName(msg->hwnd,ClassName,128);  
                if(IsWindowUnicode(msg->hwnd))  
                    goproc=GetWindowLongW(msg->hwnd,GWL_WNDPROC);  
                else 
                    goproc=GetWindowLongA(msg->hwnd,GWL_WNDPROC);  
                if(GetClassInfoW(NULL,ClassName,&WndClass))  
                {  
                    if(WndClass.lpszMenuName) {/*...............*/};  
                    if(IsWindowUnicode(msg->hwnd))  
                        WndClass.lpfnWndProc=(WNDPROC)GetClassLongW(msg->hwnd,GCL_WNDPROC);  
                }  
                //..............  
                msg->message=0;  
                msg->wParam=0;  
                msg->lParam=0;  
            }  
        }  
        else if(gfHookEnabled && !gfHookDisabled)  
        {  
            DWORD procid;  
            DWORD id=GetWindowThreadProcessId(msg->hwnd,&procid);  
            if(procid != _gpidSpyxx && id != _gpidSpyxx && GetCurrentThreadId() != _gpidSpyxx)  
                func(0,0,msg->hwnd,msg->message,msg->wParam,msg->lParam,msg->time,msg->pt.x,msg->pt.y);  
        }  
    }  
    return CallNextHookEx(_ghhkMsgHook,nCode,wParam,lParam);  
} 
```

## 结论
&emsp;&emsp;可以分析出，spy++通过windows hook方式注入相关功能到目标进程，从而实现自己的目的。Window编写hook时是要把函数放在专门的dll中，hook后所有程序都会自动加载该dll