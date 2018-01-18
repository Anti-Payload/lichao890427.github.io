---
layout: post
title: 研究星号密码查看器的工作原理
categories: Reverse_Engineering
description: 研究星号密码查看器的工作原理
keywords: 星号密码
---

## 背景

&emsp;&emsp;很久以前就有这个东西了，不过一直不知道原理是什么。这个和灰色按钮破解还有些不同，灰色按钮那个是EnumWindow和EnableWindow搞定的，而星号密码经我研究发现是通过远程线程注入API实现的。从前不会调试，静态反汇编面对海量代码任你是谁也无法那么容易找到，然而熟悉了调试以后，这个就变的简单了。主窗口有个放大镜，拖动放大镜到任意窗口就可以显示密码文本框的实际字符串，根据这一点可以下SetWindowText系列API断点。下面用WinDbg来调试：  
* bp User32!SetWindowTextA
* bp User32!SetWindowTextW
* bp User32!SetDlgItemTextA
* bp User32!SetDlgItemTextW

&emsp;&emsp;再移动放大镜到某窗口，断在了SetWindowTextA，Shift+F11跳出后进入ViewPass，输入k查看调用栈：

```Txt
0018f8e4 774b62fa ViewPass+0x1cf0
0018f910 774df943 USER32!gapfnScSendMessage+0x332
0018f98c 774df784 USER32!GetCursor+0x263
0018f9dc 774cafac USER32!GetCursor+0xa4
0018f9fc 774b62fa USER32!DrawTextExA+0xd4
0018fa28 774b6d3a USER32!gapfnScSendMessage+0x332
0018faa0 774b77c4 USER32!GetThreadDesktop+0xd7
0018fb00 774b788a USER32!CharPrevW+0x138
0018fb10 774dc81f USER32!DispatchMessageW+0xf
0018fb3c 774dcde7 USER32!IsDialogMessageW+0x11e
0018fb80 774dcf5c USER32!DialogBoxIndirectParamW+0x1f4
0018fbac 774dce8a USER32!DialogBoxIndirectParamAorW+0x108
0018fbcc 774fcb58 USER32!DialogBoxIndirectParamAorW+0x36
0018fbf8 004014f7 USER32!DialogBoxParamA+0x4c
0018fefc 00404145 ViewPass+0x14f7
0018ff88 775b338a ViewPass+0x4145
0018ff94 77d49f72 kernel32!BaseThreadInitThunk+0x12
0018ffd4 77d49f45 ntdll!RtlInitializeExceptionChain+0x63
0018ffec 00000000 ntdll!RtlInitializeExceptionChain+0x36
```

    可以看出是个回调函数，再看剩下的代码：

```Txt
00401CC7   68 C8F44000      PUSH ViewPass.0040F4C8
00401CCC   50               PUSH EAX
00401CCD   E8 9E130000      CALL ViewPass.00403070
00401CD2   59               POP ECX
00401CD3   59               POP ECX
00401CD4   8D85 D8FEFFFF    LEA EAX,DWORD PTR SS:[EBP-128]
00401CDA   50               PUSH EAX
00401CDB   68 EC030000      PUSH 3EC
00401CE0   FF75 08          PUSH DWORD PTR SS:[EBP+8]
00401CE3   FF15 B0A14000    CALL DWORD PTR DS:[<&USER32.GetDlgItem>] ; USER32.GetDlgItem
00401CE9   50               PUSH EAX
00401CEA   FF15 2CA24000    CALL DWORD PTR DS:[<&USER32.SetWindowTex>; USER32.SetWindowTextA
00401CF0   33C0             XOR EAX,EAX
00401CF2   5F               POP EDI
00401CF3   5E               POP ESI
00401CF4   5B               POP EBX
00401CF5   C9               LEAVE
```

&emsp;&emsp;发现上一句是GetDlgItem，第二个参数1004，用exescope查确实是编辑框id。执行结束再看调用栈，发现一堆USER32，一切迹象表明这个函数是个回调函数，那么有2个工作要做：
* 从栈上找到其参数②找到起始地址
* 因为API函数序言部分为push ebp;mov ebp,esp;sub esp,???; 而调用之前为push param4;push param3;push param2;push param1;call func;因此从ebp推断，d ebp得：

```Txt
0018f8e4  10 f9 18 00 fa 62 4b 77-84 0a 51 00 00 02 00 00  .....bKw..Q.....
0018f8f4  4c 0d 0b 00 ac 16 00 00-fe 14 40 00 cd ab ba dc  L.........@.....
0018f904  01 00 00 00 00 00 00 00-fe 14 40 00 8c f9 18 00  ..........@.....
0018f914  43 f9 4d 77 fe 14 40 00-84 0a 51 00 00 02 00 00  C.Mw..@...Q.....
```

&emsp;&emsp;查MSDN得知窗口回调函数格式为：  
`int WINAPI DialogProc(HWND hwndDlg,UINT uMsg,WPARAM wParam,LPARAM lParam)`  
&emsp;&emsp;[ebp]为原始ebp，[ebp+4]为call func指令下一句地址，[ebp+8]为hwndDlg=0x510a84，也就是[ebp+12]为uMsg=0x200 查winuser.h可知为WM_MOUSEMOVE
②找到起始地址需要借助调用该回调函数的系统函数，查看调用栈，地址为774df943，那么用命令 `u 0x774df943 l-10` 看一下该地址处的前一条指令

```Txt
0:000> u 774df943 l-10
USER32!GetCursor+0x253:
774df933 1cff            sbb     al,0FFh
774df935 7518            jne     USER32!GetCursor+0x26f (774df94f)
774df937 ff7514          push    dword ptr [ebp+14h]
774df93a ff7510          push    dword ptr [ebp+10h]
774df93d 56              push    esi
774df93e e89469fdff      call    USER32!gapfnScSendMessage+0x30f (774b62d7)
   课件上一条指令为call    USER32!gapfnScSendMessage+0x30f ，此时bp 774df93e重新运行，跟踪系统函数直到进入用户程序范围
774b62d7 55              push    ebp
774b62d8 8bec            mov     ebp,esp
774b62da 56              push    esi
774b62db 57              push    edi
774b62dc 53              push    ebx
774b62dd 68cdabbadc      push    0DCBAABCDh
774b62e2 56              push    esi
774b62e3 ff7518          push    dword ptr [ebp+18h]
774b62e6 ff7514          push    dword ptr [ebp+14h]
774b62e9 ff7510          push    dword ptr [ebp+10h]
774b62ec ff750c          push    dword ptr [ebp+0Ch]
774b62ef 64800dca0f000001 or      byte ptr fs:[0FCAh],1
774b62f7 ff5508          call    dword ptr [ebp+8]
```
&emsp;&emsp;走到这时，就进入了程序范围`ViewPass+0x14fe=004014fe`处，该处为函数起始位置：  

```Txt
0:000> u . l20
ViewPass+0x14fe:
004014fe 55              push    ebp
004014ff 8bec            mov     ebp,esp
00401501 81ec28040000    sub     esp,428h
00401507 8b4d0c          mov     ecx,dword ptr [ebp+0Ch]
0040150a b838010000      mov     eax,138h
0040150f 53              push    ebx
00401510 56              push    esi
00401511 3bc8            cmp     ecx,eax
00401513 57              push    edi
00401514 0f87cf030000    ja      ViewPass+0x18e9 (004018e9)
0040151a 0f847c030000    je      ViewPass+0x189c (0040189c)
00401520 8bc1            mov     eax,ecx
00401522 83e810          sub     eax,10h
00401525 0f8466030000    je      ViewPass+0x1891 (00401891)
0040152b 2d00010000      sub     eax,100h
```

&emsp;&emsp;由于第一次运行是WM_MOVEMOUSE响应，因此去掉所有断点并下条件断点：`bp 004014fe ".if poi([esp+8])=0x200 {} .else {gc}`"，这样的结果是，无论是否移动放大镜，只要鼠标滑过就会触发，结合IDA分析，得到大致代码如下：

```C++
HWND hglasswnd;//放大镜窗口  
HICON hglassicon_new,hglassicon_origin;//放大镜图标  
BOOL IsMouseDown=FALSE;  
HCURSOR hCursor;  
HINSTANCE hInstance;  
int WINAPI MainDlg(HWND hDlg,UINT uMsg,WPARAM wParam,LPARAM lParam)  
{  
   switch(uMsg)  
   {  
       case WM_INITDIALOG:  
       {  
       }  
       break;  
       case WM_LBUTTONDOWN:  
       {  
           POINT pt={LOWORD(lParam),HIWORD(lParam)};  
           HWND hobjwnd=ChildWindowFromPoint(hDlg,pt);  
           if(hobjwnd == hglasswnd)  
           {  
               SetCapture(hDlg);  
               SetCursor(hCursor);  
               SendMessage(hglasswnd,STM_SETICON,(WPARAM)hglassicon_new,NULL);  
               IsMouseDown=TRUE;  
           }  
           return 0;  
       }  
       case WM_LBUTTONUP:  
       {  
           if(IsMouseDown)  
           {  
               ReleaseCapture();  
               IsMouseDown=FALSE;  
               SetCursor(LoadCursor(hInstance,MAKEINTRESOURCE(32512)));  
               SendMessage(hglasswnd,STM_SETICON,(WPARAM)hglassicon_origin,0);  
           }  
           return 0;  
       }  
       case WM_MOUSEMOVE:  
       {  
           POINT Point;  
           TCHAR String[256],ClassName[256];  
           GetCursorPos(&Point);  
           if(!IsMouseDown)  
               return 0;  
           String[0]='\0';  
           HWND hobjwnd=WindowFromPoint(Point);//获取鼠标位置的窗口句柄  
           if(hobjwnd)  
           {  
               DWORD threadid=GetWindowThreadProcessId(hobjwnd,NULL);  
               if(threadid == GetWindowThreadProcessId(hDlg,0))  
                   return 0;//所在窗口正是密码查看器窗口  
               if(GetClassName(hobjwnd,ClassName,256))  
               {  
                   if(!strcmp(ClassName,"Internet Explorer_Server"))  
                   {  
                       POINT pt=Point;  
                       ScreenToClient(hobjwnd,&pt);  
                       htmlHWNDtoDocument(hobjwnd,pt);  
                       return 1;  
                   }  
                   strupr(ClassName);  
                   if(!strcmp(ClassName,"BUTTON") || !strcmp(ClassName,"#32770"))//#32770是对话框默认窗口类名  
                   {  
                       HWND hparent;  
                       if(!strcmp(ClassName,"#32770"))  
                           hparent=hobjwnd;  
                       else 
                           hparent=GetParent(hobjwnd);  
                       if(hparent)  
                       {  
                           POINT pt=Point;  
                           ScreenToClient(hparent,&pt);  
                           HWND first=ChildWindowFromPointEx(hparent,pt,CWP_ALL);  
                           if(first == hobjwnd)  
                           {  
                               while(first=GetWindow(first,GW_HWNDNEXT))//按z-order遍历当前鼠标位置所有窗口  
                               {  
                                   RECT rt;  
                                   GetWindowRect(first,&rt);  
                                   if(PtInRect(&rt,Point))  
                                       break;  
                               }  
                           }  
                           if(first)  
                           {  
                               hobjwnd=first;  
                               GetClassName(first,ClassName,255);  
                               _strupr(ClassName);  
                           }  
                       }  
                   }  
                   LONG style=GetWindowLong(hobjwnd,GWL_STYLE);  
                   if(!strcmp(ClassName,"EDIT") || !strcmp(ClassName,"TEDIT") ||!strcmp(ClassName,"THUNDERRTTEXTBOX") ||  
                       !strcmp(ClassName,"THUNDERRT3TEXTBOX") || !strcmp(ClassName,"THUNDERRT4TEXTBOX") ||   
                       !strcmp(ClassName,"THUNDERRT5TEXTBOX") || !strcmp(ClassName,"THUNDERRT6TEXTBOX") ||  
                       strstr(ClassName,"Pass") || strstr(ClassName,"pass"))  
                   {  
                       if(style&ES_PASSWORD/*且为Windows NT系统*/)  
                       {  
                           DWORD tid;  
                           GetWindowThreadProcessId(hobjwnd,&tid);  
                           HANDLE hproc=OpenProcess(PROCESS_QUERY_INFORMATION|PROCESS_VM_WRITE|PROCESS_VM_READ|PROCESS_VM_OPERATION|PROCESS_CREATE_THREAD,false,tid);  
                           if(hproc)  
                           {  
                               GetPassFromStar(hproc,hobjwnd,String);//远程注入到该进程以获取密码  
                               CloseHandle(hproc);  
                           }  
                           else 
                               strcpy(String,"");  
                       }  
                   }  
                   else if(!(style&(WS_POPUP|WS_CAPTION|WS_SYSMENU)))  
                       SendMessage(hobjwnd,WM_GETTEXT,255,(LPARAM)String);  
               }  
           }  
           SetWindowText(GetDlgItem(hobjwnd,1004),String);//更新编辑框字符串  
       }  
       break;  
       //....................  
   }  
} 
  // 放大镜获取网页密码代码
#pragma comment(lib, "comsuppw.lib")   
typedef HRESULT  (WINAPI* MyObjectFromLresult)(LRESULT lResult,REFIID riid,WPARAM wParam,void** ppvObject);  
void CheckPassword(HWND hDlg,IHTMLDocument2* pDoc2,POINT pt)  
{  
   if(pDoc2==NULL)  
       return ;  
   CComPtr<IHTMLElement> pElement;  
   HRESULT hr=pDoc2->elementFromPoint(pt.x,pt.y,&pElement);  
   if(SUCCEEDED(hr))  
   {     
       CComPtr<IHTMLInputTextElement> pPwdElement;  
       hr=pElement->QueryInterface(IID_IHTMLInputTextElement,(void**)&pPwdElement);  
       if(SUCCEEDED(hr))  
       {  
           CComBSTR type;  
           hr=pPwdElement->get_type(&type);  
           if(SUCCEEDED(hr))  
           {  
               if(type == _T("password"))  
               {  
                   CComBSTR pwd;  
                   hr=pPwdElement->get_value(&pwd);  
                   if(SUCCEEDED(hr))  
                   {  
                       if(pwd.Length() != 0)  
                       {  
                           LPCTSTR result;//=(_bstr_t)pwd;  
                           SetWindowText(GetDlgItem(hDlg,1004),result);  
                       }  
                   }  
               }  
           }  
       }  
   }  
}  
void htmlHWNDtoDocument(HWND hWnd,POINT pt)  
{  
    HMODULE hmod=LoadLibrary("OLEACC.DLL");  
    IHTMLDocument2* objhtml;  
    if(!hmod)  
    {  
        MessageBox(NULL,"请您安装Microsoft Active Accessibility","提示",MB_OK);  
        return;  
    }  
    if(hWnd)  
    {  
        DWORD dwResult;  
        CComPtr<IHTMLDocument2>spDoc;  
        UINT htmlmsg=RegisterWindowMessage("WM_HTML_GETOBJECT");  
        SendMessageTimeout(hWnd,htmlmsg,0,0,SMTO_ABORTIFHUNG,1000,&dwResult);  
        MyObjectFromLresult ObjectFromLresult=(MyObjectFromLresult)GetProcAddress(hmod,"ObjectFromLresult");  
        if(ObjectFromLresult != NULL)  
        {  
            LRESULT lRes=ObjectFromLresult(dwResult,IID_IHTMLDocument,0,(void**)&spDoc);  
            if(SUCCEEDED(lRes))  
            {  
                CComPtr<IDispatch> spDisp;  
                CComQIPtr<IHTMLWindow2> spWin;  
                spDoc->get_Script( &spDisp );  
                spWin=spDisp;  
                spWin->get_document( &objhtml );  
                CheckPassword(hWnd,objhtml,pt);  
            }  
        }  
    }  
    FreeLibrary(hmod);  
} 
struct ParamStruct  
{  
    HWND hWnd;//目标窗口句柄  
    FARPROC MySendMessage;  
    TCHAR Buffer[256];  
};  
_declspec(naked) DWORD WINAPI Injectbegin(ParamStruct* ps)  
{  
    _asm  
    {  
        push esi;  
        mov esi,ps;  
        lea eax,[esi+8];  
        push eax;  
        push 100h;  
        push 0Dh;  
        push [esi];  
        call [esi+4];//SendMessage(ps->hWnd,WM_GETTEXT,256,(LPARAM)ps->Bufer)  
        and byte ptr [esi+107h],0;//ps->Buffer[255]='\0'  
        pop esi;  
        retn 4;  
    }  
}  
_declspec(naked)  void Injectend()  
{//该函数只为定位Injectbegin函数结尾而已  
    _asm retn;  
}  
 void GetPassFromStar(HANDLE hProcess,HWND hWnd,LPTSTR buf,BOOL IsUnicode)  
{  
    DWORD writenum=0;  
    HANDLE hThread=NULL;  
    DWORD ThreadId=0;  
    DWORD ExitCode=0;  
    LPVOID lpParameter,lpStartAddress;  
    __try 
    {  
        HMODULE hmod=GetModuleHandle("user32");  
        if(hmod)  
        {  
            ParamStruct ps;  
            ps.hWnd=hWnd;  
            if(IsUnicode)  
                ps.MySendMessage=GetProcAddress(hmod,_T("SendMessageW"));  
            else 
                ps.MySendMessage=GetProcAddress(hmod,_T("SendMessageA"));  
            if(ps.MySendMessage)  
            {  
                //先在目标进程分配参数空间，以便把参数传给之后写入进程的代码  
                if(lpParameter=VirtualAllocEx(hProcess,NULL,sizeof(ParamStruct),MEM_COMMIT,PAGE_READWRITE))  
                {  
                    WriteProcessMemory(hProcess,lpParameter,&ps,sizeof(ParamStruct),&writenum);  
                    const int  INJECTCODESIZE =(char*)Injectend-(char*)Injectbegin;//Release版有效  
                    if(lpStartAddress=VirtualAllocEx(hProcess,NULL,INJECTCODESIZE,MEM_COMMIT,PAGE_EXECUTE_READWRITE))  
                    {  
                        WriteProcessMemory(hProcess,lpStartAddress,(LPVOID)Injectbegin,INJECTCODESIZE,&writenum);  
                        HANDLE newthread=CreateRemoteThread(hProcess,NULL,0,(LPTHREAD_START_ROUTINE)lpStartAddress,lpParameter,0,&ThreadId);  
                        if(newthread)  
                        {  
                            WaitForSingleObject(newthread,INFINITE);  
                            ReadProcessMemory(hProcess,lpParameter,&ps,sizeof(ParamStruct),&writenum);  
                            if(IsUnicode)  
                            {//Unicode处理的不好  
                                wcscpy((wchar_t*)buf,(wchar_t*)ps.Buffer);  
                            }  
                            else 
                            {  
                                strcpy((char*)buf,(char*)ps.Buffer);  
                            }  
                        }  
                    }  
                }  
            }  
        }     
    }  
    __finally 
    {  
        if(lpParameter)  
            VirtualFreeEx(hProcess,lpParameter,0,MEM_RELEASE);  
        if(lpStartAddress)  
            VirtualFreeEx(hProcess,lpStartAddress,0,MEM_RELEASE);  
        if(hThread)  
        {  
            GetExitCodeThread(hThread,&ExitCode);  
            CloseHandle(hThread);  
        }  
    }  
} 

```