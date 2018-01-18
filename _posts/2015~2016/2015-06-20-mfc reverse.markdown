---
layout: post
title: MFC逆向-消息响应函数的定位
categories: Reverse_Engineering
description: MFC逆向-消息响应函数的定位
keywords: mfc
---

# MFC逆向-消息响应函数的定位

## 背景

&emsp;&emsp;MFC-Microsoft Foundation Class，微软基础类库，他封装了Windows API以便用户更快速的开发界面功能程序。然而该库及其庞大而复杂，需要有C++的功底否则很难解决bug，逆向起来也是需要一定技巧。Windows消息以如下：

```C++
#define WM_NULL                                                        0x0000                        0
 #define WM_CREATE                                                0x0001
 #define WM_DESTROY                                                0x0002
 #define WM_MOVE                                                        0x0003
 #define WM_SIZE                                                        0x0005
 #define WM_ACTIVATE                                                0x0006
 #define WM_SETFOCUS                                                0x0007
 #define WM_KILLFOCUS                                        0x0008
 #define WM_ENABLE                       0x000A
 #define WM_SETREDRAW                    0x000B
 #define WM_SETTEXT                      0x000C
 #define WM_GETTEXT                      0x000D
 #define WM_GETTEXTLENGTH                0x000E
 #define WM_PAINT                        0x000F
// more
```

## 尝试

&emsp;&emsp;如果拿其他自己写的程序来说明显然没有说服力，我们就找一个MFC写的来看，超凡搜索BeyondSeacher ，下载下来以后主程序是P2P Searcher.exe，网上说他们抄袭了amule的，我们就来看看究竟。用peid看做初步判断，可以看到Microsoft Visual C++ 7.0 [Debug]    IDA载入，先来找入口点，Start->WinMain->可以看到了IDA识别出了Afx系内部函数，且为静态调用。对于对话框类MFC程序，C??App和C???Dlg是关键类，在源码中可以看到这一点，总是有个全局对象，例如“CtestmfcApp theApp;”，那么就需要在c库_initc()中进行初始化(__xc_a -> __xc_z)，另外C???App构造的时候会先构造父类CWinApp，且构造父类之后会设置设置虚表从而构造子类，C???Dlg过程类似。根据该原理可以定位到这2个类的位置。先找到CWinApp::CWinApp构造函数，发现索引位置：

```C++
void *__thiscall sub_401850(void *this)
 {
   void *v1; // esi@1

   v1 = this;
   CWinApp::CWinApp(this, 0);
   *(_DWORD *)v1 = &off_4315F8;
   return v1;
 }
 ```

显然是C???App的构造函数，先不急着看，往上层找：
```C++
int sub_42FF70()
 {
   sub_401850(&unk_43F0E0);
   return atexit(sub_42FFF0);
 }
```

&emsp;&emsp;可见此处执行的是c库，程序启动时构造C???App myapp，同时注册退出时的析构函数，以便清理资源，在往上看已经是一堆要在启动时要初始化的函数了，接着看sub_401850，其虚表为off_4315F8：

```Txt
.rdata:004315F8 off_4315F8      dd offset sub_42B312    ; DATA XREF: sub_401850+A o
 .rdata:004315FC                 dd offset sub_401970
 .rdata:00431600                 dd offset nullsub_4
 .rdata:00431604                 dd offset ?OnCmdMsg@CCmdTarget@@UAEHIHPAXPAUAFX_CMDHANDLERINFO@@@Z ; CCmdTarget::OnCmdMsg(uint,int,void *,AFX_CMDHANDLERINFO *)
 .rdata:00431608                 dd offset ?OnFinalRelease@CCmdTarget@@UAEXXZ ; CCmdTarget::OnFinalRelease(void)
 .rdata:0043160C                 dd offset sub_4229A6
 .rdata:00431610                 dd offset ?_Get_deleter@_Ref_count_base@std@@UBEPAXABVtype_info@@@Z_6 ; std::_Ref_count_base::_Get_deleter(type_info const &)
 .rdata:00431614                 dd offset sub_4229AC
 .rdata:00431618                 dd offset sub_4229AC
 .rdata:0043161C                 dd offset ?GetTypeLib@CCmdTarget@@UAEJKPAPAUITypeLib@@@Z ; CCmdTarget::GetTypeLib(ulong,ITypeLib * *)
 .rdata:00431620                 dd offset sub_401840
 .rdata:00431624                 dd offset sub_422A0C
 .rdata:00431628                 dd offset sub_4229BD
 .rdata:0043162C                 dd offset sub_422A06
 .rdata:00431630                 dd offset sub_4229C9
 .rdata:00431634                 dd offset sub_4229C3
 .rdata:00431638                 dd offset sub_4229FD
。。。。。。。。。。。。
```

&emsp;&emsp;根据虚函数特点，如果子类重新定义了虚函数后会覆盖父类虚函数，因此未识别出来的均是经过修改的，那么我们只需要对照MFC源码中CWinApp的类布局，就可以知道未识别出的函数是哪些了，具体操作不再赘述，C???App最重要的是InitInstance函数，为虚表第21个函数，我们来看他做了些什么：

```C++
int __thiscall sub_401880(CWinApp *this)
 {
   CWinApp *v1; // esi@1
   void *v2; // eax@1
   int v3; // eax@2
   char v5; // [sp+8h] [bp-2C8h]@1
   int v6; // [sp+2CCh] [bp-4h]@1

   v1 = this;
   InitCommonControls();
   CWinApp::InitInstance(v1);
   AfxEnableControlContainer(0);
   sub_42B8C2("应用程序向导生成的本地应用程序");
   sub_40A150(&v5, 0);
   v6 = 0;
   *((_DWORD *)v1 + 7) = &v5;
   CDialog:oModal((CDialog *)&v5);
   v2 = operator new(1u);
   LOBYTE(v6) = 1;
   if ( v2 )
     v3 = sub_401000(v2);
   else
     v3 = 0;
   LOBYTE(v6) = 0;
   j_uninit(v3);
   v6 = -1;
   sub_409E20(&v5);
   return 0;
 }
```

&emsp;&emsp;我们只来讨论和普通CWinApp构造函数不同的地方，可以得知v5是我们的C???Dlg，而下面v2 = operator new(1u)显然是某个构造函数：

```C++
BOOL CP2pSearcherApp::InitInstance()
 {
         InitCommonControls();
         CWinApp::InitInstance();
         SetRegistryKey(_T("应用程序向导生成的本地应用程序"));
         CP2pSearcherDlg dlg;
         m_pMainWnd=&Dlg;
         int nResponse=dlg.DoModal();
         if (nResponse == IDOK)
         {

         }
         else if (nResponse == IDCANCEL)
         {

         }
         dispatch* mydispatch=new dispatch;        
         mydispatch->uninit();
 }
```

下面来看CP2pSearcherDlg：

```C++
void *__thiscall sub_40A150(void *this, int a2)
 {
   v2 = this;
   CDialog::CDialog(this, 0x66u, a2);
   v17 = 0;
   *(_DWORD *)v2 = &off_431C50;
   CWnd::CWnd((char *)v2 + 116);
   *((_DWORD *)v2 + 29) = &off_43326C;
   CWnd::CWnd((char *)v2 + 196);
   *((_DWORD *)v2 + 49) = &off_4334A4;
   CWnd::CWnd((char *)v2 + 276);
   *((_DWORD *)v2 + 69) = &off_43326C;
   CWnd::CWnd((char *)v2 + 356);
   *((_DWORD *)v2 + 89) = &off_43326C;
   CWnd::CWnd((char *)v2 + 436);
   *((_DWORD *)v2 + 109) = &off_43311C;
   CWnd::CWnd((char *)v2 + 516);
   *((_DWORD *)v2 + 129) = &off_4335F4;
   *((_DWORD *)v2 + 149) = 0;
   *((_DWORD *)v2 + 156) = 15;
   *((_DWORD *)v2 + 155) = 0;
   *((_BYTE *)v2 + 604) = 0;
   LOBYTE(v17) = 7;
   v3 = sub_402BB0((char *)v2 + 628);
   *((_DWORD *)v2 + 158) = v3;
   *(_BYTE *)(v3 + 45) = 1;
   *(_DWORD *)(*((_DWORD *)v2 + 158) + 4) = *((_DWORD *)v2 + 158);
   **((_DWORD **)v2 + 158) = *((_DWORD *)v2 + 158);
   *(_DWORD *)(*((_DWORD *)v2 + 158) + 8) = *((_DWORD *)v2 + 158);
   *((_DWORD *)v2 + 159) = 0;
   LOBYTE(v17) = 8;
   v4 = sub_402BF0((char *)v2 + 640);
   *((_DWORD *)v2 + 161) = v4;
   *(_BYTE *)(v4 + 57) = 1;
   *(_DWORD *)(*((_DWORD *)v2 + 161) + 4) = *((_DWORD *)v2 + 161);
   **((_DWORD **)v2 + 161) = *((_DWORD *)v2 + 161);
   *(_DWORD *)(*((_DWORD *)v2 + 161) + 8) = *((_DWORD *)v2 + 161);
   *((_DWORD *)v2 + 162) = 0;
   v5 = (int)((char *)v2 + 656);
   *(_DWORD *)(v5 + 4) = 0;
   *(_DWORD *)(v5 + 8) = 0;
   *(_DWORD *)(v5 + 12) = 0;
   LOBYTE(v17) = 10;
   *((_DWORD *)v2 + 168) = 0;
   *((_DWORD *)v2 + 169) = 0;
   *((_DWORD *)v2 + 170) = 0;
   AfxGetModuleState();
   v6 = AfxGetModuleState();
   *((_DWORD *)v2 + 28) = LoadIconA(*((HINSTANCE *)v6 + 3), (LPCSTR)0x80);
   v7 = operator new(1u);
   LOBYTE(v17) = 11;
   if ( v7 )
     v8 = sub_401000(v7);
   else
     v8 = 0;
   LOBYTE(v17) = 10;
   *((_DWORD *)v2 + 149) = v8;
   *((_BYTE *)v2 + 684) = 0;
   v9 = operator new(0x10u);
   if ( v9 )
   {
     *(_DWORD *)v9 = 0;
     *((_DWORD *)v9 + 1) = 0;
     *((_DWORD *)v9 + 2) = 0;
     *((_BYTE *)v9 + 12) = 1;
   }
   else
   {
     v9 = 0;
   }
   *((_DWORD *)v2 + 173) = v9;
   sub_40AB90(v9);
   *((_DWORD *)v2 + 172) = 0;
   v15 = 15;
   v14 = 0;
   LOBYTE(v13) = 0;
   LOBYTE(v17) = 12;
   sub_4016A0(&unk_431DA8, 4);
   v16 = 60;
   sub_4085C0(&v12);
   sub_4016A0(&unk_431B40, 6);
   v16 = 270;
   sub_4085C0(&v12);
   sub_4016A0(&unk_431C3C, 8);
   v16 = 90;
   sub_4085C0(&v12);
   sub_4016A0("hash值", 6);
   v16 = 280;
   sub_4085C0(&v12);
   sub_4016A0(&unk_431C34, 6);
   v16 = 90;
   sub_4085C0(&v12);
   v10 = v15 < 0x10;
   *((_DWORD *)v2 + 163) = 8;
   if ( !v10 )
     j__free(v13);
   return v2;
 }
```

可以看到虚表为off_431C50：

```Txt
.rdata:00431C50 off_431C50      dd offset loc_42B8BC    ; DATA XREF: sub_409E20+21 o
 .rdata:00431C50                                         ; sub_40A150+41 o
 .rdata:00431C50                                         ; Exception filter 1 for function 401990
 .rdata:00431C54                 dd offset sub_40A440
 .rdata:00431C58                 dd offset nullsub_4
 .rdata:00431C5C                 dd offset unknown_libname_209 ; MFC 3.1-11.0 32bit
 .rdata:00431C60                 dd offset ?OnFinalRelease@CWnd@@UAEXXZ ; CWnd::OnFinalRelease(void)
 .rdata:00431C64                 dd offset sub_4229A6
 .rdata:00431C68                 dd offset ?_Get_deleter@_Ref_count_base@std@@UBEPAXABVtype_info@@@Z_6 ; std::_Ref_count_base::_Get_deleter(type_info const &)
 .rdata:00431C6C                 dd offset sub_4229AC
 .rdata:00431C70                 dd offset sub_4229AC
 .rdata:00431C74                 dd offset ?GetTypeLib@CCmdTarget@@UAEJKPAPAUITypeLib@@@Z ; CCmdTarget::GetTypeLib(ulong,ITypeLib * *)
 .rdata:00431C78                 dd offset sub_401C60
 .rdata:00431C7C                 dd offset sub_422A0C
 .rdata:00431C80                 dd offset sub_4229BD
 .rdata:00431C84                 dd offset sub_422A06
 .rdata:00431C88                 dd offset sub_423A14
 .rdata:00431C8C                 dd offset sub_4229C3
 .rdata:00431C90                 dd offset sub_4229FD
```

&emsp;&emsp;同样我们只关注很少的一些，OnInitDialog为初始化函数，GetMessageMap可以得到消息响应函数，OnInitDialog做的最重要的一件事为：
```
if ( !sub_401020(*((_DWORD *)v1 + 149)) )
     sub_4243C6("无法连入emule网络", "警告", 0);
```
&emsp;&emsp;而sub_4243C6做的是和上面uninit对应的init，都是dispatch.dll中的， 而149对应的变量则在CP2pSearcherDlg构造函数中可以找到踪迹。

```C++
 v7 = operator new(1u);
   LOBYTE(v17) = 11;
   if ( v7 )
     v8 = sub_401000(v7);
   else
     v8 = 0;
   LOBYTE(v17) = 10;
   *((_DWORD *)v2 + 149) = v8;
   *((_BYTE *)v2 + 684) = 0;
   v9 = operator new(0x10u);
   if ( v9 )
   {
     *(_DWORD *)v9 = 0;
     *((_DWORD *)v9 + 1) = 0;
     *((_DWORD *)v9 + 2) = 0;
     *((_BYTE *)v9 + 12) = 1;
   }
   else
   {
     v9 = 0;
   }
   *((_DWORD *)v2 + 173) = v9;
   sub_40AB90(v9);
   *((_DWORD *)v2 + 172) = 0;
   v15 = 15;
   v14 = 0;
   LOBYTE(v13) = 0;
   LOBYTE(v17) = 12;
   sub_4016A0(&unk_431DA8, 4);
   v16 = 60;
   sub_4085C0(&v12);
   sub_4016A0(&unk_431B40, 6);
   v16 = 270;
   sub_4085C0(&v12);
   sub_4016A0(&unk_431C3C, 8);
   v16 = 90;
   sub_4085C0(&v12);
   sub_4016A0("hash值", 6);
   v16 = 280;
   sub_4085C0(&v12);
   sub_4016A0(&unk_431C34, 6);
   v16 = 90;
   sub_4085C0(&v12);
   v10 = v15 < 0x10;
   *((_DWORD *)v2 + 163) = 8;
   if ( !v10 )
     j__free(v13);
```

&emsp;&emsp;看完了之后我们知道了真正做的是dispatch.dll中的init和uninit，下面我们再来看消息处理

```C++
void ****sub_401C60()
 {
   return &off_4316D0;
 }
 .rdata:004316D0 off_4316D0      dd offset off_432340    ; DATA XREF: sub_401C60 o
 .rdata:004316D4                 dd offset dword_4316D8
 .rdata:004316D8 dword_4316D8    dd 113h                 ; DATA XREF: .rdata:004316D4 o
 .rdata:004316DC                 dd 0
 .rdata:004316E0                 dd 0
 .rdata:004316E4                 dd 0
 .rdata:004316E8                 dd 11h
 .rdata:004316EC                 dd offset OnTimer
 .rdata:004316F0                 dd 112h
 .rdata:004316F4                 dd 0
 .rdata:004316F8                 dd 0
 .rdata:004316FC                 dd 0
 .rdata:00431700                 dd 1Bh
 .rdata:00431704                 dd offset OnSysCommand
。。。。。。。。。。。
```

## 从MFC源码找答案

&emsp;&emsp;先来看MFC里的相关知识，来说明这里为何这么做，我们每添加一个消息，都会在消息映射里增加一条，例如

```C++
BEGIN_MESSAGE_MAP(CtestmfcDlg, CDialogEx)
         ON_WM_PAINT()
         ON_WM_QUERYDRAGICON()
         ON_BN_CLICKED(IDOK, &CtestmfcDlg::OnBnClickedOk)
         ON_NOTIFY(NM_RCLICK, IDC_LIST1, &CtestmfcDlg::OnNMRClickList1)
 END_MESSAGE_MAP()

struct AFX_MSGMAP
 {
         const AFX_MSGMAP* (PASCAL* pfnGetBaseMap)();
         const AFX_MSGMAP_ENTRY* lpEntries;
 };
 struct AFX_MSGMAP_ENTRY
 {
         UINT nMessage;   // windows message
         UINT nCode;      // control code or WM_NOTIFY code
         UINT nID;        // control ID (or 0 for windows messages)
         UINT nLastID;    // used for entries specifying a range of control id's
         UINT_PTR nSig;       // signature type (action) or pointer to message #
         AFX_PMSG pfn;    // routine to call (or special value)
 };
```

## 回到分析
&emsp;&emsp;从这里很显然的可以看到如何对应上消息响应了。。。。。。。不用解释了吧，如此，对照前面说的windows消息代码，结合exescope查看控件id，可以吧004316D8开始的AFX_MSGMAP_ENTRY标注成消息回调函数：

```Txt
OnTimer
OnSysCommand
OnPaint
OnDragIcon
OnSearch
OnSelectSource
OnTcnSelChange
OnLvnColumnClick
OnNMLVRClickList
OnNMTCRClickList
OnNMLVDoubleClick
OnSize
```

